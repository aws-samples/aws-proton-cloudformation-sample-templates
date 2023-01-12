## サービス削除

デプロイしていた `front-end` サービスを削除します。

```bash
aws proton delete-service \
  --name "front-end"
```

サービスが正常に削除されるのを待ちます。

```bash
aws proton wait service-deleted --name front-end

aws proton get-service --name front-end
```

## IAM ロール削除

Proton がリソースのプロビジョニングや AWS CloudFormation スタックの管理を行うために作成した `ProtonServiceRole` を削除します。

```bash
aws iam delete-role --role-name ProtonServiceRole
```

## リファレンス

### AWS Proton

- [aws proton delete-service](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/proton/delete-service.html)
- [aws proton wait](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/proton/wait/index.html)
  - [aws proton wait service-deleted](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/proton/wait/service-deleted.html)
- [aws proton get-service](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/proton/get-service.html)

### AWS IAM

- [aws iam delete-role](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-role.html)
