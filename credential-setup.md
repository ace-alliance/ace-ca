# Setup directory

`git clone https://github.com/ace-alliance/ace-ca`

# Create CA

```
openssl genpkey -algorithm ed25519 -out ca.key
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -config openssl.config
```

# Create Keys/Certs for cc Roles

## Create Membership Keys/Certs

```
openssl genpkey -algorithm ed25519 -out member/yourname.key
openssl req -new -key member/yourname.key -out member/yourname.csr --config openssl.config
openssl x509 -req -CA ca.crt -CAkey ca.key -in member/yourname.csr -days 730 -sha256 -out member/yourname.crt
```

## Create Delegator Keys/Certs

```
openssl genpkey -algorithm ed25519 -out delegate/yourname.key
openssl req -new -key delegate/yourname.key -out delegate/yourname.csr --config openssl.config
openssl x509 -req -CA ca.crt -CAkey ca.key -in delegate/yourname.csr -days 730 -sha256 -out delegate/yourname.crt
```

## Create Voter Keys/Certs

```
openssl genpkey -algorithm ed25519 -out voter/yourname.key
openssl req -new -key voter/yourname.key -out voter/yourname.csr --config openssl.config
openssl x509 -req -CA ca.crt -CAkey ca.key -in voter/yourname.csr -days 730 -sha256 -out voter/yourname.crt
```

# Cold Credential

```
orchestrator-cli init-cold \
  --seed-input <YOURUTXOTXIN> \
  --mainnet \
  --ca-cert ca.crt \
  --membership-cert member/name1.crt \
  --membership-cert member/name2.crt \
  --membership-cert member/name3.crt \
  --delegation-cert delegate/name1.crt \
  --delegation-cert delegate/name2.crt \
  --delegation-cert delegate/name3.crt \
  -o init-cold
```

## Validate hashes of NFT (TODO)

```
openssl asn1parse < ca.key|tail -n 1|cut -d':' -f4
```

## Submit Transaction

```
cardano-cli-ng conway transaction build \
  --change-address $(< orchestrator.addr) \
  --tx-in <YOUR_TXIN> \
  --tx-in-collateral <YOUR_COLLATERAL> \
  --tx-out "$(< init-cold/nft.addr) + 5000000 + 1 $(< init-cold/minting.plutus.hash).$(< init-cold/nft-token-name)" \
  --tx-out-inline-datum-file init-cold/nft.datum.json \
  --mint "1 $(< init-cold/minting.plutus.hash).$(< init-cold/nft-token-name)" \
  --minting-script-file init-cold/minting.plutus \
  --mint-redeemer-file init-cold/mint.redeemer.json \
  --out-file init-cold/tx-mint-cc-nft.txbody

cardano-cli-ng conway transaction sign \
  --tx-file init-cold/tx-mint-cc-nft.txbody \
  --out-file init-cold/tx-mint-cc-nft.txsigned \
  --signing-key-file pay.skey

cardano-cli-ng conway transaction submit --tx-file init-cold/tx-mint-cc-nft.txsigned
```

# Hot Credential

```
orchestrator-cli init-hot \
  --seed-input <YOUR_TXIN> \
  --mainnet \
  --cold-nft-policy-id "$(< init-cold/minting.plutus.hash)" \
  --cold-nft-token-name "$(< init-cold/nft-token-name)" \
  --voting-cert voter/name1.crt \
  --voting-cert voter/name2.crt \
  --voting-cert voter/name3.crt \
  -o init-hot
```

## Validate hashes of NFT (TODO)

```
openssl asn1parse < ca.key|tail -n 1|cut -d':' -f4
```

## Submit Transaction

```
cardano-cli-ng conway transaction build \
  --change-address $(< orchestrator.addr) \
  --tx-in <YOUR_TXIN> \
  --tx-in-collateral <YOUR_COLLATERAL> \
  --tx-out "$(< init-hot/nft.addr) + 5000000 + 1 $(< init-hot/minting.plutus.hash).$(< init-hot/nft-token-name)" \
  --tx-out-inline-datum-file init-hot/nft.datum.json \
  --mint "1 $(< init-hot/minting.plutus.hash).$(< init-hot/nft-token-name)" \
  --minting-script-file init-hot/minting.plutus \
  --mint-redeemer-file init-hot/mint.redeemer.json \
  --out-file init-hot/tx-mint-cc-nft.txbody

cardano-cli-ng conway transaction sign \
  --tx-file init-hot/tx-mint-cc-nft.txbody \
  --out-file init-hot/tx-mint-cc-nft.txsigned \
  --signing-key-file pay.skey

cardano-cli-ng conway transaction submit --tx-file init-hot/tx-mint-cc-nft.txsigned
```

# Authorize Hot Credential

```
cardano-cli conway query utxo --address $(< init-cold/nft.addr) --output-json > cold-nft.utxo
orchestrator-cli authorize \
  -u cold-nft.utxo \
  --cold-credential-script-file init-cold/credential.plutus \
  --hot-credential-script-file init-hot/credential.plutus \
  --out-dir authorize
```

## Submit Transaction

```
cardano-cli-ng conway transaction build \
  --tx-in <YOUR_TXIN> \
  --tx-in-collateral <YOUR_COLLATERAL> \
  --tx-in $(cardano-cli query utxo --address $(< init-cold/nft.addr) --output-json | jq -r 'keys[0]') \
  --tx-in-script-file init-cold/nft.plutus \
  --tx-in-inline-datum-present \
  --tx-in-redeemer-file authorize/redeemer.json \
  --tx-out "$(< authorize/value)" \
  --tx-out-inline-datum-file authorize/datum.json \
  --required-signer-hash $(< delegate/name1.keyhash) \
  --required-signer-hash $(< delegate/name2.keyhash) \
  --required-signer-hash $(< delegate/name3.keyhash) \
  --certificate-file authorize/authorizeHot.cert \
  --certificate-script-file init-cold/credential.plutus \
  --certificate-redeemer-value {} \
  --change-address $(< orchestrator.addr) \
  --out-file authorize/tx-authorize-hot.txbody

cardano-cli-ng conway transaction sign \
  --tx-file init-hot/tx-mint-cc-nft.txbody \
  --out-file init-hot/tx-mint-cc-nft.txsigned \
  --signing-key-file pay.skey

cardano-cli-ng conway transaction submit --tx-file init-hot/tx-mint-cc-nft.txsigned

