---
title: "Siv3D 製アプリをビルドする GitHub Actions を書いた"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "siv3d"]
published: true
---

この記事は [Siv3D Advent Calendar 2023](https://qiita.com/advent-calendar/2023/siv3d) 19 日目の記事です。

## はじめに

Siv3D 製アプリをビルドするための GitHub Actions アクションを書き、Marketplace に公開したためそれについて書きます。

[8 日目の記事](https://qiita.com/tyanmahou/items/027aa427b9c5f67d0e0d)と被るところがかなりあります。ご了承ください。

## 今回書いたもの

https://github.com/Ryoga-exe/Siv3D-Actions

リポジトリはこちらです。[テスト用のワークフロー](https://github.com/Ryoga-exe/Siv3D-Actions/tree/main/.github/workflows) もあります。

:::message
現在 (2023-12-19 日時点) では Windows と Linux ビルドに対応しています。
:::

また、 Marketplace 上で公開しているので誰にでも簡単に、すぐ使用することができます！

https://github.com/marketplace/actions/setup-siv3d-and-build-apps

## サンプル

`main` ブランチへの push と PR をトリガーに Siv3D 製アプリを Windows ビルドし、Artifacts にアップロードするワークフローです。

```yml
name: Build Siv3D App on Windows

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

env:
  # VS ソルーションファイル (.sln) へのパス
  SOLUTION_FILE_PATH: "."
  # App フォルダのパス
  APP_PATH: "./App"

permissions:
  contents: read

jobs:
  build:
    runs-on: windows-latest

    steps:
    - uses: actions/checkout@v3

    # ビルドする
    - name: Setup Siv3D and build
      uses: Ryoga-exe/Siv3D-Actions@v2
      with:
        solution-path: ${{ env.SOLUTION_FILE_PATH }}

    # Artifacts にアップロードする
    - name: Publish App
      uses: actions/upload-artifact@v3
      with:
        name: App
        path: |
          ${{ env.APP_PATH }}
          !${{ env.APP_PATH }}/engine
          !${{ env.APP_PATH }}/AS_DEBUG
          !${{ env.APP_PATH }}/Screenshot
          !${{ env.APP_PATH }}/*.ico
          !${{ env.APP_PATH }}/Resource.rc
```

:::details Linux 向けにビルドするサンプル

```yml
name: Build Siv3D App on Linux

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

env:
  SOLUTION_FILE_PATH: "."
  SIV3D_VERSION: "0.6.13"
  APP_PATH: "."

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    # ビルドする
    - name: Setup Siv3D and build
      uses: Ryoga-exe/Siv3D-Actions@v2
      with:
        solution-path: ${{ env.SOLUTION_FILE_PATH }}

    # Artifacts にアップロードする
    - name: Publish App
      uses: actions/upload-artifact@v3
      with:
        name: App-Linux
        path: |
          ${{ env.APP_PATH }}
          !${{ env.APP_PATH }}/build
          !${{ env.APP_PATH }}/.gitignore
          !${{ env.APP_PATH }}/CMakeList.txt
          !${{ env.APP_PATH }}/*.cpp
          !${{ env.APP_PATH }}/*.hpp
          !${{ env.APP_PATH }}/*.h
```

:::

Artifacts にアップロードする際の

```
!${{ env.APP_PATH }}/engine
!${{ env.APP_PATH }}/AS_DEBUG
!${{ env.APP_PATH }}/Screenshot
!${{ env.APP_PATH }}/*.ico
!${{ env.APP_PATH }}/Resource.rc
```

の部分は [Siv3D リファレンスの 41.9 同梱する必要が無いファイル](https://zenn.dev/reputeless/books/siv3d-documentation/viewer/tutorial-release#41.9-%E5%90%8C%E6%A2%B1%E3%81%99%E3%82%8B%E5%BF%85%E8%A6%81%E3%81%8C%E7%84%A1%E3%81%84%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB) を参考に指定しています。

## 仕様

Windows/Linux ビルドはランナーによって切り替えられます。つまり、`runs-on: windows-latest` と指定すると Windows 向けのビルドが、`runs-on: ubuntu-latest` と指定すると Linux 向けのビルドが行われます。 

引数に `siv3d-version` を取ることができますが、`0.6.13` のように指定することができます。
この引数が省略された場合は、Windows 向けのビルドでは `*.vcxproj` ファイルをから自動で検出します。また Linux 向けのビルドでは最新の OpenSiv3D のバージョンを使用します。

また、[公式のデフォルトの .gitignore](https://siv3d.github.io/ja-jp/tools/gitignore/) をリポジトリで使用していることを想定しています。

## 実装上の工夫など

### .vcxproj ファイルから Siv3D のバージョンを検出する

`.vcxproj` ファイルを読んでみると、Siv3D のバージョンが含まれているようなところが見つかります。
一部抜き出したものを以下に示します。

```xml
<PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|x64'">
<LinkIncremental>false</LinkIncremental>
<OutDir>$(SolutionDir)Intermediate\$(ProjectName)\Release\</OutDir>
<IntDir>$(SolutionDir)Intermediate\$(ProjectName)\Release\Intermediate\</IntDir>
<LocalDebuggerWorkingDirectory>$(ProjectDir)App</LocalDebuggerWorkingDirectory>
<IncludePath>$(SIV3D_0_6_13)\include;$(SIV3D_0_6_13)\include\ThirdParty;$(IncludePath)</IncludePath>
<LibraryPath>$(SIV3D_0_6_13)\lib\Windows;$(LibraryPath)</LibraryPath>
</PropertyGroup>
```

そのため、これを正規表現で抜き出して検出しています。
PowerShell には正規表現をはじめとする強力な関数があり、また GitHub Actions 上で使えます。
そのため、PowerShell を用いて以下のような実装をしました。

```powershell
$vcxproj = (Get-ChildItem -Recurse *.vcxproj).FullName
$content = ""
foreach($item in $vcxproj) { $content += (Get-Content $item) }
$m = [regex]::Match($content, "SIV3D_\d*_\d*_\d*")
$siv3d_version = $m.Value -replace "^SIV3D_", "" -replace "_", "."
Write-Host "Siv3D Version is ${siv3d_version}"
```

`*.vcxproj` にマッチするファイルを列挙し、その内容から `SIV3D_\d*_\d*_\d*` にマッチする文字列を抽出しています。

### 最新の Siv3D のバージョンを取得する

Linux ビルドでは引数 `siv3d-version` が省略された際には最新のバージョンを使用します。
[Siv3D/OpenSiv3D](https://github.com/Siv3D/OpenSiv3D) の[リリースページ](https://github.com/Siv3D/OpenSiv3D/releases)を見ると、バージョンをタグで管理していることがわかります。
GitHub API を使用すれば最新のリリースを取得することができるため、この API を用いて最新のタグの名前から取得するような実装にしました。

```bash
export latest-version=$(curl --silent "https://api.github.com/repos/Siv3D/OpenSiv3D/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/' | sed -E 's/v//')
echo "Latest Siv3D Version is ${siv3d_version}"
```

### キャッシュを利用する

Windows 向けの OpenSiv3D SDK は手動インストールできるように Zip ファイルが用意されています。
今回書いたアクションでは、それをダウンロードしてきて Zip ファイルを展開し、OpenSiv3D の環境をセットアップしています。
しかし、この Zip ファイルに含まれるファイル数はとても大きく、ダウンロード及び展開する際に長い時間がかかってしまいます。
そのため、今回書いたアクションではこのファイルをキャッシュし、高速化を図っています。
キャッシュがヒットしたらそのキャッシュを用いてビルドします。

実際に書いたワークフローは以下のような感じにしています。キャッシュがヒットした際には OpenSiv3D SDK のダウンロードをスキップしています。

```yml
- name: Cache OpenSiv3D SDK
    id: cache-siv3d
    uses: actions/cache@v3
    with:
    path: "D:\\OpenSiv3D_SDK_${{ steps.siv3d-version.outputs.SIV3D_VERSION }}"
    key: opensiv3d_sdk-${{ steps.siv3d-version.outputs.SIV3D_VERSION }}

- name: Download OpenSiv3D SDK
    if: steps.cache-siv3d.outputs.cache-hit != 'true'
    working-directory: "D:\\"
    run: |
    Invoke-WebRequest -Uri "https://siv3d.jp/downloads/Siv3D/manual/${{ steps.siv3d-version.outputs.SIV3D_VERSION }}/OpenSiv3D_SDK_${{ steps.siv3d-version.outputs.SIV3D_VERSION }}.zip" -OutFile "OpenSiv3D_SDK.zip"
    7z x "OpenSiv3D_SDK.zip"
    shell: pwsh
```

また、Linux 向けのビルドについては[公式ドキュメントの方法](https://siv3d.github.io/ja-jp/download/ubuntu/#3-siv3d-%E3%82%92%E3%83%93%E3%83%AB%E3%83%89%E3%81%99%E3%82%8B)に従っています。しかし、Siv3D をビルドする関係上非常に長い時間がかかってしまいます。
そのため、ここでもキャッシュを利用し高速化を実現しています。
Siv3D のソースコード及びビルド結果をキャッシュしており、キャッシュがヒットした際にはこのキャッシュを用いてビルドします。
キャッシュしない場合は 10 分以上かかっていましたが、これを 2 分程度にまで短縮することができました。

## おわりに

今後は Web 向けのビルドへの対応や様々な環境に柔軟に対応できるようにするなど進めていきたいと思っています。

ぜひ使ってみてください！
