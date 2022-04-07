# 【Kubernetes】Ingress の host名によるパスルールの挙動を確認

## 確認したいこと

```yaml:ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
spec:
  rules:
  - host: ingress1.example.com
    http:
      paths:
      - path: /*
        backend:
          service:
            name: ingress-service1
  - host: ingress2.example.com
    http:
      paths:
      - path: /*
        backend:
          service:
            name: ingress-service2
  - host: ingress3.example.com
    http:
      paths:
      - path: /*
        backend:
          service:
            name: ingress-service3
```

↑上記のようにhost名でバックエンドのサービスをそれぞれ振り分けるIngressを作成したとして、
DNSへのレコード登録を下記のとおり行う

```zone:zonefile
ingress.example.com IN A <LBのIP>
ingress1.example.com IN CNAME ingress.example.com
ingress2.example.com IN CNAME ingress.example.com
ingress3.example.com IN CNAME ingress.example.com
```

LBへ付与したIPをAレコードで解決できるようなDNSレコードをまず登録して、
バックエンドサービス用のドメインはそれぞれ先述のAレコードに対してCNAMEレコードで解決できるように登録する

この場合、ingress.yamlで指定したhost名によるバックエンドの振り分けは正常に動作するのか?

## 懸念事項

ingressに到達する際のhost名がすべて`ingress.example.com(Aレコード)`となってしまい、正常に振り分けできないのではないかという疑問

## 検証

```zone:zonefile
ingress.example.com IN A 34.102.185.221
ingress1.example.com IN CNAME ingress.example.com
ingress2.example.com IN CNAME ingress.example.com
ingress3.example.com IN CNAME ingress.example.com
```

で登録して実際にアクセスする

→ちゃんとホスト名パスで振り分けられた！