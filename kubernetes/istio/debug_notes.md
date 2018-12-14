# debug notes


## Debugging Envoy and Pilot

* 使用`proxy-status`查看网格的状态

```
# istioctl proxy-status
PROXY                                                  CDS        LDS        EDS               RDS          PILOT                            VERSION
details-v1-876bf485f-zswkn.default                     SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
istio-egressgateway-7dc5cbbc56-4p2gn.istio-system      SYNCED     SYNCED     SYNCED (100%)     NOT SENT     istio-pilot-56bfdbffff-9ks4j     1.0.2
istio-ingressgateway-7958d776b5-wx84p.istio-system     SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
productpage-v1-8d69b45c-l6grt.default                  SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
ratings-v1-7c9949d479-6bq86.default                    SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
reviews-v1-85b7d84c56-tkn5d.default                    SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
reviews-v2-cbd94c99b-wgvfj.default                     SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
reviews-v3-748456d47b-fmq2v.default                    SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-56bfdbffff-9ks4j     1.0.2
```

参考[Get an overview of your mesh](https://istio.io/help/ops/traffic-management/proxy-cmd/#get-an-overview-of-your-mesh).


## Keys and Certificates

* 检查`Citadel`是否正常的创建了secret

```
# kubectl get secret --all-namespaces | grep istio.io/key-and-cert
default          istio.default                                        istio.io/key-and-cert                 3      25h
istio-system     istio.default                                        istio.io/key-and-cert                 3      25h
istio-system     istio.istio-citadel-service-account                  istio.io/key-and-cert                 3      25h
istio-system     istio.istio-cleanup-secrets-service-account          istio.io/key-and-cert                 3      25h
istio-system     istio.istio-egressgateway-service-account            istio.io/key-and-cert                 3      25h
istio-system     istio.istio-galley-service-account                   istio.io/key-and-cert                 3      25h
istio-system     istio.istio-ingressgateway-service-account           istio.io/key-and-cert                 3      25h
istio-system     istio.istio-mixer-service-account                    istio.io/key-and-cert                 3      25h
istio-system     istio.istio-pilot-service-account                    istio.io/key-and-cert                 3      25h
istio-system     istio.istio-security-post-install-account            istio.io/key-and-cert                 3      25h
istio-system     istio.istio-sidecar-injector-service-account         istio.io/key-and-cert                 3      25h
istio-system     istio.prometheus                                     istio.io/key-and-cert                 3      25h
```

参考[Keys and Certificates](https://istio.io/help/ops/security/keys-and-certs/).