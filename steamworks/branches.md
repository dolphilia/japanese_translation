ブランチ (ベータ)
Steamworks ドキュメント > ストアプレゼンス > アプリケーション > ブランチ (ベータ)
ブランチ (別名ベータ) とは、アプリケーションの特定のビルドで、Steamを通じて公開または非公開にできます。

ブランチは、Steamworksのアプリ管理のビルドページから管理します。 利用可能なアプリブランチの一覧には、これまでに作成したすべてのブランチとデフォルトブランチが表示されます。

available_app_branches.png

[新しいアプリブランチを作成] ボタンを使用して、新しいブランチを作成できます。 ブランチ作成の際には、スペースを一切入れず、わかりやすい名前を付けてください。 ユーザーがあなたのゲームのベータを選択する際に、この名前が表示されます。 パスワードが指定されている場合、ブランチへのアクセスには、ブランチ名とパスワードの入力が必要です。

Steamクライアント内でカスタムブランチに切り替えるには、ライブラリ内でゲームを右クリックし、[プロパティ] を選択します。 利用可能なタブの1つに「ベータ」タブがあります。
ゲームのベータは「参加希望のベータを選択してください」の下のドロップダウンメニュー内にリストされます。

select_branch.png

テスターやユーザーがブランチを選択すると、Steamクライアントはブランチのダウンロードを開始し、現在インストールされているブランチと置き換えます。
デフォルトブランチ
デフォルトブランチは、Steamの顧客に配信されるゲームのバージョンです。 これは、リリース前にテスターがデフォルトでダウンロードするバージョンです。 ゲームがリリースされると、お客様へのデフォルトビルドの配信のために、何か特別なことをする必要はありません。

注: コンテンツをアップロードする際、デフォルトブランチのビルドを自動的にライブに設定することはできません。 アプリ管理からで手動で行う必要があります。
新しいブランチの追加
様々な要件を満たすためにタイトルに追加のブランチを作成できます。 ベータブランチの最も一般的な用途はテスト用です。
タイトルがSteamでリリース済みの場合の例：
エラーレポートのポップアップに表示されたいくつかの問題を修正する必要があり、顧客向けにゲームをアップデートする前に修正をテストする手段が必要だとします。
新しいコンテンツを追加するのではないので、顧客がベータ版に参加して、問題対処中のバージョンをプレイしたいかどうかを気にする必要はありません。
設定手順は以下の通りです
アプリ管理の [ビルド] タブで新しいブランチを作成します
Steamworks SDKのContentBuilderツールを使用して、テストしたいビルドをアップロードします
アプリ管理のページを更新すると、最新の50 件のビルドを表示:というリストの一番上に最新のビルドが表示されるはずです
[ブランチ用にビルドをライブに設定...] というドロップダウンメニューからそのブランチを選択します
選択したビルドの [変更をプレビュー] ボタンをクリックしてください。
[今すぐビルドをライブに設定] をクリックすると、ブランチがすぐに Steam で利用可能になります。