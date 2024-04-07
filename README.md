# http-mitm-proxy

[![Crates.io](https://img.shields.io/crates/v/http-mitm-proxy.svg)](https://crates.io/crates/http-mitm-proxy)

A HTTP proxy server library intended to be a backend of application like Burp proxy.

- Sniff HTTP and HTTPS traffic by signing certificate on the fly.
- Server Sent Event
- WebSocket ("raw" traffic only. Parsers will not be implemented in this crate.)

## Usage

```rust, no_run
use std::path::PathBuf;

use clap::Parser;
use futures::StreamExt;
use http_mitm_proxy::MitmProxy;

#[derive(Parser)]
struct Opt {
    #[clap(requires("private_key"))]
    cert: Option<PathBuf>,
    #[clap(requires("cert"))]
    private_key: Option<PathBuf>,
}

fn make_root_cert() -> rcgen::CertifiedKey {
    let mut param = rcgen::CertificateParams::default();

    param.distinguished_name = rcgen::DistinguishedName::new();
    param.distinguished_name.push(
        rcgen::DnType::CommonName,
        rcgen::DnValue::Utf8String("<HTTP-MITM-PROXY CA>".to_string()),
    );
    param.key_usages = vec![
        rcgen::KeyUsagePurpose::KeyCertSign,
        rcgen::KeyUsagePurpose::CrlSign,
    ];
    param.is_ca = rcgen::IsCa::Ca(rcgen::BasicConstraints::Unconstrained);

    let key_pair = rcgen::KeyPair::generate().unwrap();
    let cert = param.self_signed(&key_pair).unwrap();

    rcgen::CertifiedKey { cert, key_pair }
}

#[tokio::main]
async fn main() {
    let opt = Opt::parse();

    let root_cert = if let (Some(cert), Some(private_key)) = (opt.cert, opt.private_key) {
        // Use existing key
        let param =
            rcgen::CertificateParams::from_ca_cert_pem(&std::fs::read_to_string(cert).unwrap())
                .unwrap();
        let key_pair =
            rcgen::KeyPair::from_pem(&std::fs::read_to_string(private_key).unwrap()).unwrap();

        let cert = param.self_signed(&key_pair).unwrap();

        rcgen::CertifiedKey { cert, key_pair }
    } else {
        make_root_cert()
    };

    let root_cert_pem = root_cert.cert.pem();
    let root_cert_key = root_cert.key_pair.serialize_pem();

    let proxy = MitmProxy::new(
        // This is the root cert that will be used to sign the fake certificates
        Some(root_cert),
        // This is the connector that will be used to connect to the upstream server from proxy
        tokio_native_tls::native_tls::TlsConnector::new().unwrap(),
    );

    let (mut communications, server) = proxy.bind(("127.0.0.1", 3003)).await.unwrap();
    tokio::spawn(server);

    println!("HTTP Proxy is listening on http://127.0.0.1:3003");

    println!();
    println!("Trust this cert if you want to use HTTPS");
    println!();
    println!("{}", root_cert_pem);
    println!();
    println!("Private key");
    println!("{}", root_cert_key);

    /*
        Save this cert to ca.crt and use it with curl like this:
        curl https://www.google.com -x http://127.0.0.1:3003 --cacert ca.crt
    */

    while let Some(comm) = communications.next().await {
        let uri = comm.request.uri().clone();
        // modify the request here if you want
        let _ = comm.request_back.send(comm.request);
        if let Ok(Ok(mut response)) = comm.response.await {
            let mut len = 0;
            let body = response.body_mut();
            while let Some(frame) = body.next().await {
                if let Ok(frame) = frame {
                    if let Some(data) = frame.data_ref() {
                        len += data.len();
                    }
                }
            }
            println!(
                "{}\t{}\t{}\t{}",
                comm.client_addr,
                uri,
                response.status(),
                len
            );
        }
    }
}

```