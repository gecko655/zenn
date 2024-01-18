---
title: "1password に登録したSSH 鍵が EC2 Windows インスタンスで使えなくなったかと思った話"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["1Password", "SSH", "AWS", "windows", "ec2"]
published: true
publication_name: "mixi"
---

1password に登録してチーム共有して満足していた SSH 鍵が、 EC2 Windows インスタンスのログイン時に使えなくなったかと思い冷や汗をかいた話を書きます。

## OpenSSL 7.8 以降では秘密鍵の形式が変わった
先に前提知識を書きます。

この記事では深く説明しませんが、OpenSSL 7.8からデフォルトの秘密鍵の形式が代わり、
`-----BEGIN RSA PRIVATE KEY-----` から始まる形式から
`-----BEGIN OPENSSH PRIVATE KEY-----` に変更されました。

旧形式の RSA 鍵が欲しい場合は `ssh-keygen -m PEM` のように `-m PEM` のオプションを付ける必要があります。

詳しくは以下の記事を参照してください。
https://dev.classmethod.jp/articles/openssh78_potentially_incompatible_changes/

旧形式 <-> 新形式の変換についてはこの記事が詳しい
https://qiita.com/angel_p_57/items/6e826105d50cbb0e0abe


## 1password の SSH 鍵は OpenSSH 7.8 以降の鍵に勝手に変換される

1password では SSH 鍵を登録しておくことができます。

https://developer.1password.com/docs/ssh/manage-keys/

1password は、 OpenSSH 7.8 以前の旧形式の秘密鍵を登録すると、 OpenSSH 7.8以降の新形式にデフォルトで変換されて出力する機能があるようです。
例えば、以下の旧形式 RSA 秘密鍵のことを考えます。
（**このキーペアはこの記事の解説用に作ったもので、実際に使用しているものではありません。**）

旧形式秘密鍵
```
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAl5TDfXutwkKpKQyrMtKsUJ4txah1Wa4zB0qPWwTDxXI3JiRP
MbVHj//YMDsHvJgOUrnQmlm5qklcAxmIGRaQeqLwN82Hk+bkz8JiTw5fhNN09wLZ
h5oPpL2g0YtJkDez/h7QOGG+AUeqHc3BhPUYw/AmZ0Vz2Hgu+902iPm63qx2FbaJ
XWgDYACYW0VExlkB+27zb3rcf5PmGbsxEDN+YrIWAOdQMjuL4COBvgyEF3pQ5VWr
cdHL0G+cBOdMGMkofe4yNWoHjBezHOiImngdJ+BSQ24H/ePNGx+xTNPar+4GMbaY
bmU6V3fVL5GaFXB5tAikoUtK/KKAaG44W7i1sQIDAQABAoIBABWwwfCY3BpqM46e
M3lIUEQQ/nfETVaR6NdoQe9CVUlRuLaKh5KTYDQo5iSwrcC8+X+4+zK7GChX6wS+
iI0ef3otCrbRgE8XwTJBnJO3eM+m+pvGGp6e2xI12GdjyRkL/4OjoSQQSIIiYSN2
p/SAphSTrsskUxlsyTxdQUqEgcsPJiYNYvY0qizE4vIec2NmXnLgELOtEFVyFlf8
y+e32NjN9h0ZCKq1BzgHBpSWCLkhl6CIZ0C9ucpcK7Doyr7jy70oWGMibVamnNYo
eYd2kqxDJ08lOUomhcalzteTQ0KPxile7nI6i0Vu+4EG5ahok2dTjBU1fGf69vGj
AhoyeeECgYEA1DgIHJivt6oA+YtK2zWMzRt82hHpjRrQWneaQCLGaaC8rMSuhNJm
xrKGpplJ1kyBW5OZ66ZIrKvjR3QUij/hIztulSKf+uglHpLiahui124bwfUeyJvk
0Z+XezEJ2WJUe3uaswsaVs5XNv0NL26HxpIkyvXoAMZAn8KpBKIawCUCgYEAttpB
Jp+2z69BFq1verZ5t2Eo+r6JSMYKzH28mMuxT3wpvlI/TDpHCphlQsbyr//eBZXe
QndgbueswkYfeUxOpMNJr26K7HQ7bNOxSSLyCmloTvJNDUfnghWA9stP2fG9xhSV
plpB1uy0DkhKQNm7OfyhNHEXW4is6+62Mf/rs50CgYB6l9+/vUiM/e1QOvJETdwH
xKBMTVqww9Om3z7BXBVogY1c9MWoPu9WS11TsmugG1QC9fJN2iJTdXx3E4ymDJ7f
Pn70Mitew2pmDg4zo8FfV+E7G4Hr+3qkyd+1L6/z30TUjKPiWECf8tUZE/fg9aYD
xPryMDoU8HH2mHoRDiAL7QKBgDenSEskM3UU51+qnBKidXtuFBX1Zj2DIYhKANwU
qzwBE4d86w0dc7/y0Gc5vGX7H61dhw993BkFZJyg0TWPFySo18WQhLIhUnD2IbCb
9UVb/caBkxgmuXzrZJw5F23DWTpvy3idYgqzcr4iHI+OdaDZlosqnKxcdh09Q7EG
Lsw1AoGAOx+wn8dPN3ZxDrZ/qDxsk0vfUQuLS9I7I4DZomchjz+MOnx6Ct6Ij+a0
Nmgfb1D/09VQoG/ZilZd/W62mZi4pXzqsRZNOcTChyZwMCCT1QpbTJB3RuyNQR5Y
0gL68K2qDKlWzfFwlcghU4I62hpGlfQPir6OvNE33KnXHIncXqE=
-----END RSA PRIVATE KEY-----
```

公開鍵
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCXlMN9e63CQqkpDKsy0qxQni3FqHVZrjMHSo9bBMPFcjcmJE8xtUeP/9gwOwe8mA5SudCaWbmqSVwDGYgZFpB6ovA3zYeT5uTPwmJPDl+E03T3AtmHmg+kvaDRi0mQN7P+HtA4Yb4BR6odzcGE9RjD8CZnRXPYeC773TaI+brerHYVtoldaANgAJhbRUTGWQH7bvNvetx/k+YZuzEQM35ishYA51AyO4vgI4G+DIQXelDlVatx0cvQb5wE50wYySh97jI1ageMF7Mc6IiaeB0n4FJDbgf9480bH7FM09qv7gYxtphuZTpXd9UvkZoVcHm0CKShS0r8ooBobjhbuLWx
```

この秘密鍵を 1password に登録するとこうなります。
- 1password 登録時は秘密鍵さえ登録しておけば、公開鍵は勝手に生成してくれる。

![](/images/1password-ssh-key-ec2-windows/1.png)


その後、秘密鍵を 1password からダウンロードすると、登録した秘密鍵とは全く異なる、新形式の秘密鍵がダウンロードされます。

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAl5TDfXutwkKpKQyrMtKsUJ4txah1Wa4zB0qPWwTDxXI3JiRPMbVH
j//YMDsHvJgOUrnQmlm5qklcAxmIGRaQeqLwN82Hk+bkz8JiTw5fhNN09wLZh5oPpL2g0Y
tJkDez/h7QOGG+AUeqHc3BhPUYw/AmZ0Vz2Hgu+902iPm63qx2FbaJXWgDYACYW0VExlkB
+27zb3rcf5PmGbsxEDN+YrIWAOdQMjuL4COBvgyEF3pQ5VWrcdHL0G+cBOdMGMkofe4yNW
oHjBezHOiImngdJ+BSQ24H/ePNGx+xTNPar+4GMbaYbmU6V3fVL5GaFXB5tAikoUtK/KKA
aG44W7i1sQAAA7i4QyZfuEMmXwAAAAdzc2gtcnNhAAABAQCXlMN9e63CQqkpDKsy0qxQni
3FqHVZrjMHSo9bBMPFcjcmJE8xtUeP/9gwOwe8mA5SudCaWbmqSVwDGYgZFpB6ovA3zYeT
5uTPwmJPDl+E03T3AtmHmg+kvaDRi0mQN7P+HtA4Yb4BR6odzcGE9RjD8CZnRXPYeC773T
aI+brerHYVtoldaANgAJhbRUTGWQH7bvNvetx/k+YZuzEQM35ishYA51AyO4vgI4G+DIQX
elDlVatx0cvQb5wE50wYySh97jI1ageMF7Mc6IiaeB0n4FJDbgf9480bH7FM09qv7gYxtp
huZTpXd9UvkZoVcHm0CKShS0r8ooBobjhbuLWxAAAAAwEAAQAAAQAVsMHwmNwaajOOnjN5
SFBEEP53xE1WkejXaEHvQlVJUbi2ioeSk2A0KOYksK3AvPl/uPsyuxgoV+sEvoiNHn96LQ
q20YBPF8EyQZyTt3jPpvqbxhqentsSNdhnY8kZC/+Do6EkEEiCImEjdqf0gKYUk67LJFMZ
bMk8XUFKhIHLDyYmDWL2NKosxOLyHnNjZl5y4BCzrRBVchZX/Mvnt9jYzfYdGQiqtQc4Bw
aUlgi5IZegiGdAvbnKXCuw6Mq+48u9KFhjIm1WppzWKHmHdpKsQydPJTlKJoXGpc7Xk0NC
j8YpXu5yOotFbvuBBuWoaJNnU4wVNXxn+vbxowIaMnnhAAAAgDsfsJ/HTzd2cQ62f6g8bJ
NL31ELi0vSOyOA2aJnIY8/jDp8egreiI/mtDZoH29Q/9PVUKBv2YpWXf1utpmYuKV86rEW
TTnEwocmcDAgk9UKW0yQd0bsjUEeWNIC+vCtqgypVs3xcJXIIVOCOtoaRpX0D4q+jrzRN9
yp1xyJ3F6hAAAAgQDUOAgcmK+3qgD5i0rbNYzNG3zaEemNGtBad5pAIsZpoLysxK6E0mbG
soammUnWTIFbk5nrpkisq+NHdBSKP+EjO26VIp/66CUekuJqG6LXbhvB9R7Im+TRn5d7MQ
nZYlR7e5qzCxpWzlc2/Q0vbofGkiTK9egAxkCfwqkEohrAJQAAAIEAttpBJp+2z69BFq1v
erZ5t2Eo+r6JSMYKzH28mMuxT3wpvlI/TDpHCphlQsbyr//eBZXeQndgbueswkYfeUxOpM
NJr26K7HQ7bNOxSSLyCmloTvJNDUfnghWA9stP2fG9xhSVplpB1uy0DkhKQNm7OfyhNHEX
W4is6+62Mf/rs50AAAAAAQID
-----END OPENSSH PRIVATE KEY-----
```
以上の2つの形式の SSH 鍵はどちらも公開鍵とペアとなっており、普段 SSH 等を利用する限りでは問題は起こりません。

```bash
# 公開鍵と秘密鍵が一致することの確認
$ diff rsa_key_for_article.pub <(ssh-keygen -y -f rsa_old_format_key_for_article.pem)

$ diff rsa_key_for_article.pub <(ssh-keygen -y -f rsa_new_format_key_for_article.key)

# どちらも差がない
```

## EC2 Windows インスタンスでは旧形式 RSA 秘密鍵しか受け付けてくれない
EC2 Windows インスタンスでは、 Windows のユーザーパスワードを取得する際に、インスタンス立ち上げ時に設定した秘密鍵を送信する必要があります。

![](/images/1password-ssh-key-ec2-windows/2.png)

> Windows インスタンスでは、プライベートキーを使用して管理者パスワードを取得し、RDP を使用してログインします。

https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/ec2-key-pairs.html
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html#connect-rdp

実際に Windows インスタンスを立ててパスワードを取得する手順を見てみます。

AWS のキーペアを RSAを AWS コンソールのキーペア作成画面で作成すると、この記事を書いている時点では **OpenSSH 7.8 以前の旧形式で出力されます** 。

- この記事を書いている時点で、 Windows インスタンスでは 2048-bit SSH-2 RSA キーのみがサポートされ、 ED25519 キーはサポートされていません。
- ちなみに、前の節で解説に利用したキーも、AWS コンソールのキーペア作成画面で作成したものです。

作成した RSA 鍵を設定した Windows インスタンスでパスワードの取得画面に 旧形式の RSA 鍵をそのまま入力すると、Windows ログインのためのパスワードを得ることができます。

![](/images/1password-ssh-key-ec2-windows/3.png)

しかし、ここで旧形式秘密鍵を 1password に登録し、新形式に勝手に変換された秘密鍵を 1password から取り出した上でWindows インスタンスでパスワードの取得画面に入力すると、

> Private key must begin with "-----BEGIN RSA PRIVATE KEY-----" and end with "-----END RSA PRIVATE KEY-----"

と言われて怒られてしまいます。 EC2 Windows インスタンスのパスワード取得時は、旧形式の RSA 秘密鍵しか受け付けてくれないようです。

![](/images/1password-ssh-key-ec2-windows/4.png)


## 解決方法
旧形式秘密鍵を1passwordに登録したあとで旧形式秘密鍵が必要になった場合は以下のように解決する必要があります。

### 旧形式秘密鍵を CLI で 1password から取り出す
アプリ版やブラウザ版の1password では新形式の秘密鍵しか取り出せませんが、1password CLI では **なぜか** 旧形式の秘密鍵を取り出せます。
https://developer.1password.com/docs/cli/

`op read 'op://[保管庫名]/[アイテム名]/private_key'` のようなコマンドを実行すると、旧形式で秘密鍵を取り出すことができます。

```bash
$ op read 'op://private/rsa_key_for_article/private_key'
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAl5TDfXutwkKpKQyrMtKsUJ4txah1Wa4zB0qPWwTDxXI3JiRP
MbVHj//YMDsHvJgOUrnQmlm5qklcAxmIGRaQeqLwN82Hk+bkz8JiTw5fhNN09wLZ
# 以下略
```

- 「旧形式で取り出せる」というより、「登録した鍵をそのまま取り出せる」のかも…？（ちゃんと調べてない）
- リファレンスのコマンド例では `op read "op://app-prod/ssh key/private key?ssh-format=openssh"` となってるんですが、 `private key` ではなく `private_key` が正しそう。
    - https://developer.1password.com/docs/cli/reference/commands/read/
    - https://megalodon.jp/2024-0117-2047-06/https://developer.1password.com:443/docs/cli/reference/commands/read/
- https://developer.1password.com/docs/cli/ssh-keys/ や https://developer.1password.com/docs/cli/secrets-reference-syntax/#ssh-format-parameter にはまた全然違うことが書いてある……(アイテムへのアドレスが違う) `op read "op://Private/ssh keys/ssh key/private key?ssh-format=openssh"` ???
    - https://megalodon.jp/2024-0117-2053-29/https://developer.1password.com:443/docs/cli/ssh-keys/
- また、 `ssh-format=openssh` を付けると謎のエラー ”unsupported key type "RSA PRIVATE KEY" passed with the PEM" が発生する。付けなければ旧形式で出力される。
- `ssh-format` オプションの `openssh` 以外の選択肢はドキュメントのどこを探しても書いてない。
- この記事を書いている時点での最新バージョンは 2.24.0 です。
    - https://app-updates.agilebits.com/product_history/CLI2

参考にした記事
https://1password.community/discussion/131541/
↑ ”unsupported key type "RSA PRIVATE KEY" passed with the PEM" なるエラーが出ることも報告されている。
https://1password.community/discussion/139136/

### 秘密鍵を自分で新形式から旧形式に変換する
わざわざ 1password の CLI をインストールして 1password の機能で旧形式の鍵を取り出さなくても、新形式の鍵は `ssh-keygen -p -N "" -m pem -f [秘密鍵ファイル]` のようなコマンドで旧形式に変換できます。
- 既存のファイルを上書きする形での変換となりますのでご注意ください。

参考：  https://qiita.com/angel_p_57/items/6e826105d50cbb0e0abe


## まとめ
- RSA 鍵には OpenSSH 7.8以前の形式と7.8以降の形式がある
- 1password は旧形式の RSA 鍵を登録すると、（CLI経由以外では）新形式の鍵しか取り出せなくなる
- EC2 Windows インスタンスのパスワード取得時は旧形式の秘密鍵しか受け付けない
- 1password の CLI の SSH 鍵周りの機能がなんかバグってる OR ドキュメントが間違っているっぽいので、直してほしい


1password と EC2 Windows インスタンスの組み合わせのせいで余計な苦労をしました。なんとかしてほしい。
