# 案件EC2削除

案件用のEC2インスタンスと関連リソースを削除します。

## 使い方

```
/project:delete-project クライアント名
```

## 引数

- `$ARGUMENTS`: クライアント名（例: acme）

## 実行前確認

```
==========================================
⚠️ 案件削除の確認
==========================================

以下のリソースを削除します：
- EC2インスタンス: ar-dev-${ARGUMENTS}-server
- セキュリティグループ: ar-dev-${ARGUMENTS}-sg
- キーペア: ar-dev-${ARGUMENTS}-key
- IAMロール: ar-dev-${ARGUMENTS}-role
- ローカルSSH鍵: ~/.ssh/ar-dev-${ARGUMENTS}-key.pem
- エイリアス: ${ARGUMENTS}

この操作は取り消せません。
続行しますか？（はい/いいえ）
==========================================
```

## 削除手順

「はい」の場合、以下を実行：

### 1. EC2インスタンス削除

```bash
CLIENT_NAME="${ARGUMENTS}"
REGION="ap-northeast-1"

# インスタンスID取得
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

if [ "$INSTANCE_ID" != "None" ] && [ -n "$INSTANCE_ID" ]; then
  # スポットリクエストをキャンセル（スポットインスタンスの場合）
  SPOT_REQUEST_ID=$(aws ec2 describe-instances \
    --instance-ids ${INSTANCE_ID} \
    --query 'Reservations[0].Instances[0].SpotInstanceRequestId' \
    --output text \
    --region ${REGION})
  
  if [ "$SPOT_REQUEST_ID" != "None" ] && [ -n "$SPOT_REQUEST_ID" ]; then
    aws ec2 cancel-spot-instance-requests \
      --spot-instance-request-ids ${SPOT_REQUEST_ID} \
      --region ${REGION}
    echo "スポットリクエストキャンセル完了: ${SPOT_REQUEST_ID}"
  fi
  
  # EBSボリュームID取得（削除用）
  VOLUME_ID=$(aws ec2 describe-instances \
    --instance-ids ${INSTANCE_ID} \
    --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId' \
    --output text \
    --region ${REGION})

  # インスタンス終了
  aws ec2 terminate-instances --instance-ids ${INSTANCE_ID} --region ${REGION}
  echo "EC2終了中: ${INSTANCE_ID}"
  aws ec2 wait instance-terminated --instance-ids ${INSTANCE_ID} --region ${REGION}
  echo "EC2終了完了"
  
  # EBSボリューム削除
  if [ "$VOLUME_ID" != "None" ] && [ -n "$VOLUME_ID" ]; then
    aws ec2 delete-volume --volume-id ${VOLUME_ID} --region ${REGION}
    echo "EBSボリューム削除完了: ${VOLUME_ID}"
  fi
fi
```

### 2. セキュリティグループ削除

```bash
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ar-dev-${CLIENT_NAME}-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text \
  --region ${REGION})

if [ "$SG_ID" != "None" ] && [ -n "$SG_ID" ]; then
  aws ec2 delete-security-group --group-id ${SG_ID} --region ${REGION}
  echo "セキュリティグループ削除完了"
fi
```

### 3. キーペア削除

```bash
aws ec2 delete-key-pair --key-name ar-dev-${CLIENT_NAME}-key --region ${REGION}
echo "キーペア削除完了"
```

### 4. IAMロール削除

```bash
ROLE_NAME="ar-dev-${CLIENT_NAME}-role"
PROFILE_NAME="ar-dev-${CLIENT_NAME}-profile"

# インスタンスプロファイルからロールを削除
aws iam remove-role-from-instance-profile \
  --instance-profile-name ${PROFILE_NAME} \
  --role-name ${ROLE_NAME} 2>/dev/null

# インスタンスプロファイル削除
aws iam delete-instance-profile --instance-profile-name ${PROFILE_NAME} 2>/dev/null

# インラインポリシー削除
aws iam delete-role-policy --role-name ${ROLE_NAME} --policy-name ec2-self-stop-policy 2>/dev/null

# ロール削除
aws iam delete-role --role-name ${ROLE_NAME} 2>/dev/null

echo "IAMロール削除完了"
```

### 5. ローカルファイル削除

```bash
# SSH鍵削除
rm -f ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem

# エイリアス削除
sed -i "/alias ${CLIENT_NAME}=/d" ~/.project_aliases
source ~/.project_aliases

echo "ローカルファイル削除完了"
```

## 完了報告

```
==========================================
🗑️ 案件削除完了
==========================================

削除されたリソース:
- EC2: ar-dev-${CLIENT_NAME}-server
- セキュリティグループ: ar-dev-${CLIENT_NAME}-sg
- キーペア: ar-dev-${CLIENT_NAME}-key
- IAMロール: ar-dev-${CLIENT_NAME}-role
- SSH鍵: ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem
- エイリアス: ${CLIENT_NAME}

【残作業】
tmux windowの削除:
  window内で exit を実行

==========================================
```
