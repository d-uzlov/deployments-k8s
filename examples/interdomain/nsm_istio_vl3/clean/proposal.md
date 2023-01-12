


We need to know the following:

- ip of istiod
- istiod certificate
- service account token

We need to generate the following resources:

- mesh.yaml
- cluster.env
- workloadgroup.yaml
- destination rule
- service entry

We need the following changes:

      hostAliases:
      - ip: "172.16.0.2"
        hostnames:
        - istiod.istio-system.svc

we need to modify the following environment variables:

        - name: PROV_CERT
          value: /etc/certs
        - name: OUTPUT_CERTS
          value: /etc/certs
        - name: POD_NAMESPACE
          value: vm-ns
        - name: SERVICE_ACCOUNT
          value: serviceaccountvm

We need to mount volumes in istio-proxy:

        - mountPath: /etc/certs/
          name: etc-certs
        # - mountPath: /var/run/secrets/istio/root-cert.pem
        - mountPath: /etc/certs/root-cert.pem
          name: cert
          subPath: root-cert.pem
        - mountPath: /var/lib/istio/envoy/cluster.env
          name: cert
          subPath: cluster.env
        - mountPath: /etc/istio/config/mesh
          name: cert
          subPath: mesh.yaml
        - mountPath: /var/run/secrets/tokens/istio-token
          name: cert
          subPath: istio-token

Disable default mounts in istio-proxy:

        # - mountPath: /var/run/secrets/istio
        #   name: istiod-ca-cert
        # - mountPath: /var/run/secrets/tokens
        #   name: istio-token
        # - mountPath: /etc/istio/pod
        #   name: istio-podinfo

Mount volumes in pod:

      volumes:
      - name: cert
        configMap:
          name: cert-file
          defaultMode: 0664
      - name: etc-certs

Disable default mounts:

      # - downwardAPI:
      #     items:
      #     - fieldRef:
      #         fieldPath: metadata.labels
      #       path: labels
      #     - fieldRef:
      #         fieldPath: metadata.annotations
      #       path: annotations
      #   name: istio-podinfo
      # - name: istio-token
      #   projected:
      #     sources:
      #     - serviceAccountToken:
      #         audience: istio-ca
      #         expirationSeconds: 43200
      #         path: istio-token
      # - configMap:
      #     name: istio-ca-root-cert
      #   name: istiod-ca-cert

