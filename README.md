ガイド付き演習: GitLab CI/CD パイプラインと AAP の統合
===================================================

この演習では、GitLab と Red Hat Ansible Automation Platform の統合に基づいて、GitLab CE で PoC の CI/CD パイプラインを構築して運用します。

1. 静的解析(code review): ansible lintを使用した構文・スタイルチェックをCI pipelineで自動化します
1. テスト：playbookをテスト環境で動作確認します
1. デプロイ：テストが成功したPlaybookを自動で本番環境用のブランチにマージし、Playbookを実行します。


前提条件
-------

1. GitLab runnerのインストール

   GitLab公式リポジトリを使ってGitLab Runnerをインストールします。
   作業場所はworkstationを例に説明します。
   
   ```
   [student@workstation ~]$ curl -LO https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh
   ...output omitted...
   [student@workstation ~]$ sudo bash ./script.rpm.sh
   ...output omitted...
   [student@workstation ~]$ sudo dnf install gitlab-runner-13.12.0-1
   ...output omitted...
   ```

1. GitLab runnerの登録

   Runnerの登録は、RunnerとGitLabインスタンスをバインドするプロセスです。
   以下の登録コマンドを実行します。

   ```
   [student@workstation ~]$ sudo gitlab-runner register --non-interactive \
      --url "https://git.lab.example.com/" --registration-token "b7Gb-Q-nN_jVrtqVYR3y" \
      --executor "shell" --locked=false
   ...output omitted...
   ```

   "Runner registered successfully."とメッセージが出たか確認します。
   出ない場合は登録に失敗している可能性があるためインストラクターに確認を取ります。

1. Automation Controller CLI(awx)のインストール

   AAPをコマンドラインで操作するためのawxをインストールします。
   GitLab CIでのAAPの操作の自動化に使用します。

   ```
   [student@workstation ~]$ sudo dnf install --enablerepo=ansible-automation-platform-2.2-for-rhel-8-x86_64-rpms automation-controller-cli
   ...output omitted...
   ```

1. SSHキーの設置

   gitlab-runner ユーザーによるリモートリポジトリへのアクセスを許可するため、SSH鍵認証を有効にします。
   今回は一時的にstudentのものを借用します。

   ```
   [student@workstation ~]$ sudo cp -pr /home/student/.ssh /home/gitlab-runner/
   [student@workstation ~]$ sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.ssh
   ```

演習用ファイルの配置
------------------
workstationにプロジェクト格納用ディレクトリを作成し、このリポジトリをcloneします。

```
[student@workstation ~]$ mkdir ~/git-repos
[student@workstation ~]$ cd git-repos
[student@workstation git-repos]$ git clone https://github.com/root-inf/gitlab-cicd-aap-sample.git
...output omitted...
```

GitLab プロジェクトの作成
------------------------

Webブラウザでgit.lab.example.comにstudentでログインし、プロジェクト「gitlab_cicd_aap」を作成します。
Visibility は Internalを選択し、Initialize repository with a README にチェックを入れて作成します。


プロジェクトのクローン
--------------------

新規作成したプロジェクトをローカルにcloneします。

```
[student@workstation git-repos]$ git clone git@git.lab.example.com:student/gitlab_cicd_aap.git
...output omitted...
```

初期ファイルの配置
----------------
GitHubからクローンした gitlab-cicd-aap-sample にある以下のファイルを、GitLabからクローンしたgitlab_cicd_aapの直下にコピーします。

- playbook.yml: 初期に配置するPlaybook(対象ノードに対してpingモジュールのみ実行)
- hosts-prod: 商用環境を意図したインベントリ(servera,serverbを対象)
- hosts-test: テスト環境を意図したインベントリ(serverc,serverdを対象)

```
[student@workstation git-repos]$ cd gitlab-cicd-aap-sample
[student@workstation gitlab-cicd-aap-sample]$ cp playbook.yml hosts-prod hosts-test ../gitlab_cicd_aap/
```


最初のコミット、プッシュ
---------------------
コピーしたファイルをGitLabのリモートプロジェクトにプッシュし、AAPから利用可能にします。

```
[student@workstation gitlab-cicd-aap-sample]$ cd ../gitlab_cicd_aap/
[student@workstation gitlab_cicd_aap]$ git config --global user.name "student"
[student@workstation gitlab_cicd_aap]$ git config --global user.email "student@example.com"
[student@workstation gitlab_cicd_aap]$ git status
...output omitted...
[student@workstation gitlab_cicd_aap]$ git add .
[student@workstation gitlab_cicd_aap]$ git commit -m "init"
...output omitted...
[student@workstation gitlab_cicd_aap]$ git push origin master
...output omitted...
```

git.lab.example.comにアクセスし、コピーしたファイルがmasterブランチに存在することを確認します。


開発ブランチの作成
----------------

開発用のbranch `dev` を新規作成、スイッチします。

```
[student@workstation gitlab_cicd_aap]$ git switch -c dev
```

devブランチにスイッチできていることを確認します。

```
[student@workstation gitlab_cicd_aap]$ git branch -a
* dev
  master
...output omitted...
```

最初のコミット、プッシュ
---------------------
masterブランチと同様、GitLabのリモートプロジェクトにプッシュし、AAPから利用可能にします。

```
[student@workstation gitlab_cicd_aap]$ git push origin dev
...output omitted...
```

git.lab.example.comにアクセスし、コピーしたファイルがmasterブランチに存在することを確認します。


AAPの設定
--------

Automation Controller サーバー `controller.lab.example.com` に `admin` でログインし、Playbook実行のためのリソースを設定します。

1. 認証情報の作成

  以下の認証情報を作成します。
  設定項目・方法は3章も参考にしてください。

    - Git Project Credential
      ```
      Name: Git Project Credential
      Organization: Default
      Credential Type: Source Control
      Username: student
      SCM Private Key: /home/student/.ssh/gitlab_rsa ファイルの内容

    - Devops Machine Credential
      ```
      Name: Devops Machine Credential
      Organization: Default
      Credential Type: Machine
      Username: devops
      SCM Private Key: /home/student/.ssh/lab_rsa ファイルの内容


1. プロジェクトの作成

   以下の2つのプロジェクトを作成します。
   設定項目・方法は3章も参考にしてください。
   Credentialsは3章と同じものを設定してください。

   - CI/CD Test Project
     ```
     Name: CI/CD Test Project
     Source Control Type: Git
     Source Control URL: git@git.lab.example.com:student/gitlab_cicd_aap.git
     Source Control Branch: dev
     Source Control Credential: Git Project Credential
     Update Revision on Launch, Allow Branch Override にチェック
     ```

   - CI/CD Prod Project
     ```
     Name: CI/CD Prod Project
     Source Control Type: Git
     Source Control URL: git@git.lab.example.com:student/gitlab_cicd_aap.git
     Source Control Branch: master
     Source Control Credential: Git Project Credential
     Update Revision on Launch, Allow Branch Override にチェック
     ```

1. インベントリーの作成

   以下の2つのインベントリーを作成します。
   設定項目・方法は3章も参考にしてください。

   - CI/CD Test Inventory
     ```
     Name: CI/CD Test Inventory
     Inventory Source:
       Name: CI/CD Test Inventory Source
       Source: Sourced from a Project
       Project: CI/CD Test Project
       Inventory file: host-test
       Overwrite, Overwrite variables, Update on project update にチェック
     ```

   - CI/CD Prod Inventory
     ```
     Name: CI/CD Prod Inventory
     Inventory Source:
       Name: CI/CD Test Inventory Source
       Source: Sourced from a Project
       Project: CI/CD Prod Project
       Inventory file: host-prod
       Overwrite, Overwrite variables, Update on project update にチェック
     ```

1. ジョブテンプレートの作成

   以下の2つのジョブテンプレートを作成します。
   設定項目・方法は3章も参考にしてください。

   - Deploy Test WebServers
     ```
     Name: Deploy Test WebServers
     Inventory: CI/CD Test Inventory
     Project: CI/CD Test Project
     Playbook: playbook.yml
     Credentials: Devops Machine Credential
     ```

   - Deploy Prod WebServers
     ```
     Name: Deploy Prod WebServers
     Inventory: CI/CD Prod Inventory
     Project: CI/CD Prod Project
     Playbook: playbook.yml
     Credentials: Devops Machine Credential
     ```

1. 手動での実行確認

  作成したジョブテンプレートをそれぞれ実行し、正常に動作することを確認します。


Playbookの更新とGitLab CI用ファイルの配置
---------------------------------------

GitHubからクローンした gitlab-cicd-aap-sample にある playbook_overwrite.yml でplaybook.ymlを上書きし、Playbookに変更を加えます。
playbook_overwrite.yml では、Web serverのセットアップを行うタスクが記述されています。

```
[student@workstation gitlab_cicd_aap]$ cp ../gitlab-cicd-aap-sample/playbook_overwrite.yml ./playbook.yml
```

次に、GitLab CI設定ファイルの .gitlab-ci.yml も同様にコピーします。

```
[student@workstation gitlab_cicd_aap]$ cp ../gitlab-cicd-aap-sample/.gitlab-ci.yml ./
```


.gitlab-ci.ymlの確認
-------------------

GitLab プロジェクトに .gitlab-ci.yml ファイルが含まれている場合、GitLab はプロジェクトの CI/CD パイプラインを作成します。

このファイルは、一連のステージを定義し、各ステージは 1 つ以上のジョブを実行します。ステージは、ジョブ実行の順序を制御します。特定のジョブが失敗した場合、後続のステージとジョブは実行されません。

GitLab パイプラインジョブは、使用可能な GitLab Runner インスタンス上で実行されます。CI/CD パイプラインを任意のマシン上で実行するように GitLab Runner を設定できます。GitLab Runner の設定は、このコースの範囲外です。

教室の環境では、GitLab サーバー と Runner を使用して、シンプルな bash コマンドを実行して CI/CD パイプラインを実装します。

このプロジェクトの CI/CD パイプラインを定義している .gitlab-ci.yml ファイル(このREADMEと同梱されている)を確認します。

このパイプライン設定ファイルは、3 つのステージ (lint、deploy、auto_merge) を持つパイプラインを定義します。

lint ステージには 1 つのジョブが含まれており、syntax check and linting という名前が付いています。このジョブでは、すべての YAML ファイルに対して ansible-lint コマンドが実行されます。リポジトリーに YAML ファイルが存在しない場合、ジョブは失敗します。

deploy ステージでは、AAP ジョブテンプレートの実行がトリガーされます。dev ブランチについては Deploy Test Web Servers ジョブテンプレートが実行され、master ブランチについては Deploy Prod Web Servers テンプレートが実行されます。

auto_merge ステージでは、dev ブランチのジョブのみが定義されます。このジョブは、dev ブランチの変更を master ブランチにマージし、リモートリポジトリーにプッシュします。


subuid, subgidの設定
-------------------

以下のコマンドを実行し、 ユーザー gitlab-runner namespace用の UID、GIDを設定します。

```
[student@workstation gitlab_cicd_aap]$ echo "gitlab-runner:231072:65536" | sudo tee -a /etc/subuid
[student@workstation gitlab_cicd_aap]$ echo "gitlab-runner:231072:65536" | sudo tee -a /etc/subgid
```


環境変数の設定
-------------

GitLabのプロジェクトにCI実行用の環境変数を設定します。
プロジェクト「gitlab_cicd_aap」の Settings > CI/CD に移動し、Variables の項目で以下を設定します。

| Key | Value | Protected | Masked |
| ----|-------|-----------|------- |
| TOKENID | student | off | off |
| SECRET | *** | off | off |
| CONTROLLER_HOST | https://controller.lab.example.com | off | off |
| CONTROLLER_USERNAME | admin | off | off |
| CONTROLLER_PASSWORD | *** | off | off |
| CONTROLLER_VERIFY_SSL | false | off | off |
| GIT_REPO | git@git.lab.example.com:student/gitlab_cicd_aap.git | off | off |

パスワードは当日インストラクターから案内します。


リモートへのプッシュ
------------------

ローカルで作成したplaybook, .gitlab-ci.ymlをリモートにプッシュします。

```
[student@workstation gitlab_cicd_aap]$ git add .
[student@workstation gitlab_cicd_aap]$ git commit -m "Implemented CI pipeline"
[student@workstation gitlab_cicd_aap]$ git push origin dev
...output omitted...
```



CI pipeline の動作確認
---------------------

git.lab.example.comにstudentでログインし、CI pipeline の実行結果を確認します。

左のメニューから「CI/CD」を選択すると、右ペインに実行済 or 実行中のpipelineが表示されます。

成功すると「passed」になりますが、今回配置したplaybookではsyntax check and linting または launch test job をパスできずfailedになっているはずです。

結果をドリルダウンすると実行結果のログが表示されるので、内容を確認してください。

launch test jobの結果のログが出ていれば、その内容に従いplaybookを修正後、再度プッシュしてCI pipelineが成功するまで確認してください。
再度プッシュするときは、編集したファイルを`git add`コマンドで追加するのを忘れないようにしてください。
現状どのファイルがstageされているかを確認するには`git status`を使います。

playbook以外の要因でfailしている場合はインストラクターに連絡してください。

EOF