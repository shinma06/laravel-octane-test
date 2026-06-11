# FrankenPHP Homebrew インストール手順（macOS Apple Silicon）

> 対象: macOS (arm64) / Homebrew 導入済み / `php` (non-ZTS) インストール済み想定

---

## 1. インストール

### 1-1. Tap を追加して信頼

```bash
brew tap dunglas/frankenphp
brew trust dunglas/frankenphp
brew tap shivammathur/php
brew trust shivammathur/php
```

### 1-2. php-zts を先にインストール（リンクはしない）

```bash
brew install shivammathur/php/php-zts
brew unlink php-zts   # 通常の php と競合するためリンクしない
```

### 1-3. FrankenPHP をインストール

```bash
brew install frankenphp
```

> **既知バグ**: bottle ファイル名が `1.12.4` でも中身が `1.12.3` の場合がある。
> その場合は以下の手順で手動修正する。

---

## 2. bottle バグ発生時の手動修正

`Error: /opt/homebrew/Cellar/frankenphp/1.12.4 is not a directory` が出た場合。

### 2-1. キャッシュの bottle ファイル名を確認

```bash
ls ~/Library/Caches/Homebrew/downloads/ | grep frankenphp
```

### 2-2. 手動展開 + symlink

```bash
cd /opt/homebrew/Cellar/frankenphp
tar -xzf ~/Library/Caches/Homebrew/downloads/<上で確認したファイル名>
# 展開されるのは 1.12.3 ディレクトリ
ln -s 1.12.3 1.12.4
brew link frankenphp
```

### 2-3. @@HOMEBREW_PREFIX@@ を実際のパスに置換

```bash
for lib in \
  "@@HOMEBREW_PREFIX@@/opt/freetds/lib/libsybdb.5.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/gettext/lib/libintl.8.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/openssl@3/lib/libssl.3.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/openssl@3/lib/libcrypto.3.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/pcre2/lib/libpcre2-8.0.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/sqlite/lib/libsqlite3.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/gd/lib/libgd.3.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/gmp/lib/libgmp.10.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/icu4c@78/lib/libicuio.78.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/icu4c@78/lib/libicui18n.78.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/icu4c@78/lib/libicuuc.78.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/oniguruma/lib/libonig.5.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/unixodbc/lib/libodbc.2.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/libpq/lib/libpq.5.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/libsodium/lib/libsodium.26.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/argon2/lib/libargon2.1.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/libzip/lib/libzip.5.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/php-zts/lib/libphp.dylib" \
  "@@HOMEBREW_PREFIX@@/opt/watcher/lib/libwatcher-c.0.dylib"; do
  actual="${lib/@@HOMEBREW_PREFIX@@//opt/homebrew}"
  install_name_tool -change "$lib" "$actual" /opt/homebrew/Cellar/frankenphp/1.12.3/bin/frankenphp
done
```

### 2-4. アドホック再署名（Gatekeeper 対策）

`install_name_tool` がコード署名を無効化するため必須。

```bash
codesign --force --deep --sign - /opt/homebrew/Cellar/frankenphp/1.12.3/bin/frankenphp
```

---

## 3. 動作確認

```bash
frankenphp version
# FrankenPHP v1.12.3 (Homebrew) PHP 8.5.x Caddy v2.x.x ...
```

---

## 4. PHP コマンドの切り替え

FrankenPHP は内部で `php-zts` を使う。CLI の `php` コマンドは別途管理する。

### php-zts を使う（FrankenPHP 開発時）

```bash
brew link --overwrite php-zts
php --version  # ZTS ビルドになる
```

### 通常の php に戻す

```bash
brew link --overwrite php
php --version  # non-ZTS に戻る
```

---

## 5. 完全アンインストール

### 5-1. FrankenPHP 本体

```bash
brew unlink frankenphp
brew uninstall frankenphp
sudo rm -rf /opt/homebrew/Cellar/frankenphp
```

### 5-2. php-zts

```bash
brew unlink php-zts
brew uninstall php-zts
```

### 5-3. FrankenPHP 専用依存ライブラリ（任意）

他パッケージが使っていない場合のみ削除する。`brew uses` で確認してから実行。

```bash
for pkg in watcher freetds; do
  if [ -z "$(brew uses --installed $pkg)" ]; then
    brew uninstall $pkg
  fi
done
```

### 5-4. Tap を削除

```bash
brew untap dunglas/frankenphp
brew untap shivammathur/php  # 他に shivammathur/php 経由のパッケージがなければ
```

### 5-5. キャッシュ削除

```bash
rm -rf ~/Library/Caches/Homebrew/downloads/*frankenphp*
rm -rf ~/Library/Caches/Homebrew/downloads/*php-zts*
brew cleanup
```

### 5-6. 通常の php を再リンク

```bash
brew link --overwrite php
php --version  # 元の non-ZTS が復元されていることを確認
```

---

## 補足: 依存関係マップ

```
frankenphp
├── php-zts (shivammathur/php)   ← ZTS ビルドの PHP
├── watcher                       ← ファイル監視
├── freetds                       ← MS SQL Server クライアント
├── fontconfig / aom / libavif    ← 画像処理系
├── capstone                      ← 逆アセンブラ（デバッグ用）
├── libnghttp3 / libngtcp2        ← HTTP/3 (QUIC)
├── sqlite                        ← SQLite
└── (openssl, pcre2, gd, ... は既存 php と共有)
```
