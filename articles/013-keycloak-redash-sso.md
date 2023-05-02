---
title: "RedashをKeycloakを使ってシングルサインオン(SAML認証)できるようにする"
emoji: "🧑‍🎓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Redash','SAML','Keycloak']
published: true
---

BIツールとして今まで社内では別のツールを使っていましたが、今回Redashを使うことになり、社内のKeycloakと連携してSAMLによるSSOを実現しました。ネットには他にも書いている人はいるのですが、バージョンの差異で設定方法がかなり違っており苦戦したので（Keycloakは全然違う）、設定手順を記載します。

## 対象バージョン

* Redash : 21.0.1
* Keycloak: 11.0.0

## 最終形

以下のように、redashのログイン画面にSAML Loginが追加され、任意のIdPによるログインが可能になる。

![login page](/images/013/redash_login_page.png)

## Keycloakの設定

前提としてrealmの追加、ログインユーザーの追加はできていることとします。

### クライアントの追加

#### Create client

最初にクライアントを追加します。

管理の「クライアント」を選んで、「create client」を押します。

![create client](/images/013/keycloak_add_client_01.png)

#### クライアントの設定

次の画面ではクライアントタイプに「SAML」を選び、クライアントIDを「redash」とします（クライアントID名は任意）。

![client type](/images/013/keycloak_add_client_02.png)

クライアントの各設定は以下の通りです。

![client type](/images/013/keycloak_add_client_03.png)

「IDP-Initiated SSO URL name」の部分が、他の記事では書かれていなかった部分です。本当はこのURLはRedashが自動的に読み取ってSSOするはずなので設定は不要だと思われるのですが、私の環境ではうまく生きませんでした。そこでこの値を指定することで「Target IDP initialted SSO URL」を生成させ、Redash側で指定できるようにしました。

![client type](/images/013/keycloak_add_client_04.png)

![client type](/images/013/keycloak_add_client_05.png)

### クライアント・スコープの追加

次にクライアントスコープを追加し、Redash側にKeycloakのユーザー名などの情報を渡すようにします。

![create client](/images/013/keycloak_add_client_scope_01.png)

![create client](/images/013/keycloak_add_client_scope_02.png)

#### Mapperの追加

クライアント・スコープのMapperを追加していきます。

##### surnameとgivenNameを追加

KeycloakがもつsurnameとgivenNameを、redashのLastNameとFirstNameに紐付ける設定します。

![create client](/images/013/keycloak_add_client_scope_03.png)

![create client](/images/013/keycloak_add_client_scope_04.png)

![create client](/images/013/keycloak_add_client_scope_05.png)

![create client](/images/013/keycloak_add_client_scope_06.png)

##### emailを追加

同様にEmailの紐付け設定をします。

![create client](/images/013/keycloak_add_client_scope_07.png)

![create client](/images/013/keycloak_add_client_scope_08.png)

![create client](/images/013/keycloak_add_client_scope_09.png)

##### Audienceの追加

この設定が他の記事ではまったく触れられていない部分で、わかるまでに苦戦しました。
redash側で指定されたAudienceかどうかをチェックする処理が走るので、これを設定しないと認証でエラーになります。

![create client](/images/013/keycloak_add_client_scope_10.png)

![create client](/images/013/keycloak_add_client_scope_11.png)

![create client](/images/013/keycloak_add_client_scope_12.png)

## Redashの設定

### keycloakで必要な情報を取得する

redashの設定をする前に、設定する値をKeycloakから取ってきます。
レルムの設定から「SAML 2.0アイデンティティー・プロバイダ・メタデータ」に飛んでください。

![redash](/images/013/keycloak_redash_01.png)

必要なのは「entityId」「X509Certificate」の部分になります。

![redash](/images/013/keycloak_redash_02.png)

### redashのAuthenticationを設定する

redashの設定画面よりGeneralを選び、Authentication部分を記述します。

![redash](/images/013/keycloak_redash_03.png)

* SAML Enabled : Enabled（Static）を選択  
  ここをEnabled（Dynamic）にする記事が多いのですが、うまく処理されなかったためStaticにしています
* SAML Single Sign-on URL :「Target IDP initialted SSO URL」のURL部分
* SAML Entity ID : 上で取得したentityId
* SAML x509 : 上で取得したX509Certificate

## 確認

これでSAML認証の準備が終わったので、実際にログインしてください！
OpenIDConnectやOAuthに対応した製品であれば、比較的簡単にKeycloakでSSOできるのですが、SAMLはたいへんでした。
