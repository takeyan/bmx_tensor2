以前にも[BluemixのDockerコンテナ上でTensorFlowを動かす手順](http://qiita.com/takeyan/items/7956ec65e9cca367f8bd)を書いたことがあるのですが、一年も経たないうちに時代遅れの役立たずになってしまいました。
BluemixがKubernetesクラスタ環境の提供を開始し、今後はこれに一本化されるようなので、Kubernetesを前提に全面改訂しました。手順は以下のとおりです。

1. Kubernetesクラスターを作成する
2. CLI環境をセットアップする
3. TensorFlow用Pod定義ファイルを作成する
4. TensorFlowイメージをデプロイする

------

## 1. Kubernetesクラスターを作成する

Bluemixアカウント（トライアルアカウントでOK）はすでに持っている前提で話を進めます。本記事では次のアカウントを使用しています。

+ アカウント名（IBMid）：bmx01@takeyan.xyz
+ 組織名：bmx01
+ スペース名：Dallas

まずBluemixのポータル（https://bluemix.net ）にログインし、カタログのページに移動して、左ペインの全てのカテゴリーから「コンテナー」を選択します。
「Kubernetes Cluster」の方をクリックして、作成画面に入ります。
<img width="1280" alt="Screen Shot 2017-09-14 at 9.30.02.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/fe11dd47-549f-12bb-0202-976a0b9ff603.png">

作成画面でそのまま「クラスターの作成」をクリックします。クラスター名はデフォルトのmyclusterで構いません（好みで変更して下さい）。
<img width="1280" alt="Screen Shot 2017-09-14 at 9.33.09.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/11b5c773-823a-b823-3fee-5f2f724fa681.png">

画面が切り替わりますが、まだクラスターは作成中です。「デプロイ中」の表示が「」に変わったら完了ですが、結構時間がかかるようです（経験上は10分程度）。
ここでクラスター作成は放置して、CLIのセットアップに進むことにします。
<img width="1280" alt="Screen Shot 2017-09-14 at 9.35.20.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/bb10322d-a580-8428-2e75-f518b63b27e7.png">




## 2. CLI環境をセットアップする
### 2-1. Bluemix CLIをインストールする
最新のインストーラーを[こちらのページ](https://clis.ng.bluemix.net/ui/home.html)からダウンロードして実行して下さい。
MacやLinuxユーザーで、ターミナルからコマンド一発でインストールしたい方は以下のコマンドを実行して下さい。

```bash:macOS
sh <(curl -fsSL https://clis.ng.bluemix.net/install/osx)
```

```bash:Linux
sh <(curl -fsSL https://clis.ng.bluemix.net/install/linux)
```

Windowsでもコマンドによるインストールが可能です。手順の詳細は[こちら](https://console.bluemix.net/docs/cli/reference/bluemix_cli/index.html#install_bluemix_cli)を参照して下さい。

### 2-2. コンテナサービス用プラグインをインストールする

Bluemix CLIがインストールできたら、Bluemixのコンテナサービス用プラグインを追加でインストールします。以下のコマンドを実行して下さい（Mac,Linux,Windows共通です）。

```bash
bx plugin install container-service -r Bluemix
bx plugin list
```

以下にWindows環境での実行例を示します。
<img width="1127" alt="Screen Shot 2017-09-14 at 11.05.42.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/185a609e-3e69-ebdf-ccec-33d34f49b031.png">


手順の詳細は[こちら](https://console.bluemix.net/docs/containers/cs_cli_install.html#cs_cli_install)を参照して下さい。


### 2-3. Kubernetes CLIをインストールする

Kubernetes CLIのインストール手順は[こちらのページ](https://kubernetes.io/docs/tasks/tools/install-kubectl/)に説明があります。
また[こちらのページ](https://console.bluemix.net/docs/containers/cs_cli_install.html#cs_cli_install)のプラグインインストール手順の直後の③④にも説明があります（こちらの方が日本語なのでわかりやすいかも知れません）。
以下はKubernetesのWebサイトのインストール手順説明のページです。該当するOS環境のタブをクリックして手順を確認して下さい。
<img width="1280" alt="Screen Shot 2017-09-14 at 10.12.09.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/4e016924-b5fe-62b8-f3bf-fca4a0ae593b.png">

MacおよびLinuxの場合はそれぞれターミナルで次のコマンドを実行します。

```bash:macOS
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/darwin/amd64/kubectl
chmod +x ./kubectl 
sudo mv ./kubectl /usr/local/bin/kubectl 
```

```bash:Linux
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl 
sudo mv ./kubectl /usr/local/bin/kubectl 
```

Windowsはkubectl.exeファイル（コマンドそのもの）がダウンロードされるので、これをPATHの通っているフォルダに置いて下さい。Bluemix CLIのフォルダ（C:\Program Files\IBM\Bluemix\bin）でもいいと思います。
kubectlを引数なしで実行して、引数の一覧が表示されたらOKです。



### 2-4. kubectlコマンド実行環境をセットアップする

kubectlコマンドで、Bluemix上のKubernetesクラスター環境に接続して操作するための環境をセットアップします。手順は[こちら](https://console.bluemix.net/docs/containers/cs_cli_install.html#cs_cli_install)の「IBM Bluemix Container Service でクラスターに対して kubectl コマンドを実行するように CLI を構成する 」以下を参照して下さい。
Bluemix CLI（bxコマンド）でBluemixにログインして、一連の手順を実行して下さい。以下の例では米国南部リージョンにログインしています。他のリージョンで組織とスペースを作成した場合は、該当リージョンのAPIエンドポイントを指定して下さい。

```bash
bx login -a api.ng.bluemix.net
```

Emailアドレス（Bluemixアカウント名）とパスワードを聞かれるので、適宜応答して下さい。
アカウントの選択の要求には、数字で（おそらく選択肢は1だけだと思いますが）応答して下さい。
ログインしたらtarget名をセットして下さい。

```bash
bx target -o 組織名 -s スペース名
```

ここまでの例を以下に示します。
<img width="1038" alt="Screen Shot 2017-09-14 at 12.14.22.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/41d58406-fa2e-a769-6d24-41ce453cff7b.png">

続いてコンテナサービスの環境をセットアップします。ここでクラスター名は、Bluemixポータル画面でKubernetesクラスターを作成した際のクラスター名です。デフォルトのままならmyclusterです。

```bash
bx cs init
bx cs cluster-config クラスター名
```

bx cs cluster-configコマンドを実行すると、応答としてKUBECONFIG環境変数をセットするコマンドが表示されるので、これをコピペしてそのまま実行します。コマンド内のymlファイル名はKubernetesクラスターによって異なるので、過去に保存したコマンドではなく、必ずその都度bx cs cluster-configコマンドの応答で得られたコマンドを実行して下さい。

```bash:macOS/Linux
export KUBECONFIG=/Users/takey/.bluemix/plugins/container-service/clusters/mycluster/kube-config-・・・
```

```text:Windows
SET KUBECONFIG=C:\Users\takey\.bluemix\plugins\container-service\clusters\mycluster\kube-config-・・・
```

実行後は確認のためkubectl get nodesコマンドを実行します。

```bash
kubectl get nodes
```

ここまでの実行例を以下に示します。
<img width="1069" alt="Screen Shot 2017-09-14 at 13.35.27.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/a7792fcd-b5b1-42a6-76a5-d0c63a2346ec.png">

Kubernetesクラスターの作成がまだ実行中のタイミングでkubectl get nodesコマンドを実行するとエラー応答が返ります。下図のようにBluemixポータル画面で準備完了になってから再実行して下さい。
<img width="1280" alt="Screen Shot 2017-09-14 at 13.45.53.png" src="https://qiita-image-store.s3.amazonaws.com/0/32083/12b4849a-4394-7019-784e-f80fb7ae55fb.png">




## 3. TensorFlow用Pod定義ファイルを作成する

kubernetesクラスタにデプロイするコンテナは、podという単位で管理されます。コンテナをpodで包んでデプロイするような感覚です。
podはYAMLという書式で定義を記述し、kubectlコマンドにこの定義を読ませてデプロイします。
以下にpod定義の例を示します。

```yaml:my-tensor-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: my-tensor-pod
  labels:
    app: my-tensor
spec:
  containers:
    - name: tensorflow
      image: tensorflow/tensorflow:latest
      env:
      - name: PASSWORD
        value: "passw0rd"
```

"image:tensorflow/tensorflow:latest"のところで、DockerHubに公開されているTensorFlowのイメージを呼び出す指定をしています。<br>
TensorFlowが起動してブラウザでアクセスする際にはまずログイン画面が表示されるのですが、そこで自分で決めたパスワードを使えるようにしておきます。
そのためにenv:以下の3行で"passw0rd"というパスワードを指定しています。安全のため推測しにくいパスワードに置き換えておいて下さい。

定義はテキストファイルに保存します。ここではmy-tensor-pod.ymlというファイルに保存したとして手順を続けます。



## 4. TensorFlowイメージをデプロイする

先に作成したmy-tensor-pod.ymlを使ってpodをデプロイし、それをサービスとして公開することによりIPアドレス／ポート番号を割り当てます。
以下の手順を実行します。

```bash:podのデプロイ
kubectl create -f my-tensor-pod.yml
```

pod "my-tensor-pod" createdという応答が返れば成功です。


```bash:podの確認
kubectl get pods
```

my-tensor-svcという名前でサービスを公開します。以下のコマンドを実行します。

```bash:サービスの公開
kubectl expose pods my-tensor-pod --type=NodePort --port=8888 --name=my-tensor-svc
```

公開されたサービスを確認します。サービスに割り当てられたポート番号が表示されるのでメモしておいて下さい。

```bash:サービスの確認
kubectl get services
```

ノード一覧を表示して、外部向けIPアドレスを確認します。

```bash:ノード一覧
kubectl get nodes
```

これでTensorFlowを使用する準備ができました。先に確認したIPアドレス、ポート番号をWebブラウザに入力して、TensorFlow（厳密にはjupyter notebookサーバ）にログインします。ログイン画面が表示されたら、あらかじめpod定義に指定しておいたパスワードを入力して下さい。

jupyter notebook一覧の画面が表示されたら成功です。


以上です。



