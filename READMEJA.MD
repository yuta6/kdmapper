offsets.iniの内容
[ntoskrnl.exe]
MmAllocateIndependentPagesEx=11094668
MmFreeIndependentPages=3505200
PiDDBLock=16296992
PiDDBCacheTable=16618656
MmSetPageProtection=5223792

[ci.dll]
g_KernelHashBucketList=901264
g_HashCacheLock=266272

[WdFilter.sys]
MpBmDocOpenRules=146200
MpFreeDriverInfoEx=433976

## レジストリーで脆弱ドライバーを許可するコマンド
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CI\Config" /v VulnerableDriverBlocklistEnable /t REG_DWORD /d 0 /f

##　ビルドコマンド
MSBuild kdmapper.sln /p:Configuration=Release /p:Platform=x64
MSBuild kdmapper.sln /p:Configuration=Release_Pdb_Offset /p:Platform=x64
MSBuild kdmapper.sln /p:Configuration=lib-Release /p:Platform=x64
MSBuild kdmapper.sln /p:Configuration=lib-Release_Pdb_Offsets /p:Platform=x64

## 実行コマンド
kdmapper_Release_Pdb_Offset.exe ApDriver.sys --dontUpdateOffsets --offsetsPath "offsets.ini"
kdmapper_Release.exe ApDriver.sys --offsetsPath "\offsets.ini"



# KDMapper

KDMapperは、**iqvw64e.sys Intelドライバー**を利用して、署名されていないドライバーを手動でメモリにマッピングするシンプルなツールです。

**Windows 10 1607** から **Windows 11 24H2 26100.3915** まででテスト済み \:heavy\_check\_mark:

**注意:** すべてのコンソール出力を削除するには、定義 **DISABLE\_OUTPUT** を追加してください。

**注意:** HelloWorldの例のように、カスタムのエントリーポイントを設定することを推奨します。

---

## パラメーター:

```
--copy-header  コマンドラインでドライバーヘッダーのコピーオプションを有効にします
--free  割り当てたメモリを自動で解放します
--indPages  割り当てられた独立したページにマッピングします
--PassAllocationPtr  割り当てポインタを最初のパラメータとして渡します
[PDBオフセットベースのビルドのみ]:
--offsetsPath "FilePath"  独自のオフセットファイルパスを含めます（デフォルトは .\offsets.ini）（FilePath にスペースが含まれる場合は引用符で囲む必要があります）
--dontUpdateOffsets  オフセットファイルを更新せずに実行します（警告: 現在のWindowsビルドに対してオフセットが古くなっていないことを自分で確認する必要があります。そうでないとBSODのリスクがあります）
```

---

## 機能:

```
/GS- でコンパイルされたドライバーで動作します
どこにでも存在し、ほとんど呼び出されない NtAddAtom をフックします
MmUnloadedDrivers をクリアします
PiDDBCacheTable をクリアします
g_KernelHashBucketList をクリアします
Wdfilter RuntimeDriverList、RuntimeDriverCount、RuntimeDriverArry をクリアします
NtLoadDriver および NtUnloadDriver を使って痕跡を減らします
\Device\Nal が存在する場合はロードを防止します（BSOD防止）
ドライバーをカーネルにコピーする際、ヘッダーセクションをスキップします
ドライバーエントリー呼び出し前にパラメータを変更する機能を追加
PDB_OFFSETSマクロにより、オフセットPDB機能を利用可能（ターゲットビルドを選択）
新しいプロジェクト「SymbolsFromPDB」を導入し、.PDBファイルを解析してKDMapperが迅速にWindowsアップデートへ適応可能に（この機能にはPDB_OFFSETSマクロの定義が必要）
```

---

## ビルド方法:

* Visual Studio をインストール: [https://visualstudio.microsoft.com/downloads/](https://visualstudio.microsoft.com/downloads/)
* Windows SDK をインストール: [https://developer.microsoft.com/windows/downloads/windows-sdk/](https://developer.microsoft.com/windows/downloads/windows-sdk/)
* Windows WDK をインストール: [https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk](https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk)
* ソリューションを開く
* ターゲットビルド（例: Release）を選択
* ビルド

---

## 要件:

* ドライバーエントリーからできるだけ早く戻ること。予期しない呼び出しやPatch Guardを防ぐためです。絶対にエントリーポイント内で無限ループを作らないでください。コードを走らせ続けるにはスレッドや他の手法を使ってください（kdmapperが終了できない場合、やり方が間違っています）。
* 脆弱なドライバーリストが有効になっている場合は無効にしてください: [https://support.microsoft.com/en-au/topic/kb5020779-the-vulnerable-driver-blocklist-after-the-october-2022-preview-release-3fcbe13a-6013-4118-b584-fcfbc6a09936](https://support.microsoft.com/en-au/topic/kb5020779-the-vulnerable-driver-blocklist-after-the-october-2022-preview-release-3fcbe13a-6013-4118-b584-fcfbc6a09936)

---

## よくある問題

### シンプルなドライバーでBSOD

DriverEntryでは、**DriverObjectとRegistryPathはNULL**です。何も指定しない限り！これは手動マッピングドライバーであり、**通常の読み込み手順ではありません**。

---

### エラー \Device\Nal はすでに使用されています。

これは、intelドライバーがすでにロードされているか、他のkdmapperのインスタンスが実行中か、kdmapperがクラッシュして前回のドライバーをアンロードしなかったことを意味します。

他にkdmapperのインスタンスが実行中でないと確信している場合は、PCを再起動して修正できるか試してください。

問題が解決しない場合、intelドライバーを手動でアンロードできます（kdmapperでロードされた場合はランダムな名前で %temp% にあります）。そうでなければ、ドライバー名は iqvw64e.sys でネットワークドライバーとともにインストールされています。

---

### エラー 0xC0000022 および 0xC000009A:

どちらも通常、FACEIT ACやその他のアンチウイルス/アンチチートによるものです。

**注意:** 一部のアンチチートはゲームが開かれていなくても動作し続けます。

**注意:** 一部のアンチウイルスは無効にしてもブロックし続ける場合があります。

---

### エラー 0xC0000603:

証明書が脆弱としてブロックされ、マッパーは STATUS\_IMAGE\_CERT\_REVOKED ステータスを返します。詳細は [Microsoft](https://support.microsoft.com/en-au/topic/kb5020779-the-vulnerable-driver-blocklist-after-the-october-2022-preview-release-3fcbe13a-6013-4118-b584-fcfbc6a09936) を参照。

脆弱なドライバーリストを無効にしたい場合は regedit.exe を開き、HKEY\_LOCAL\_MACHINE\SYSTEM\CurrentControlSet\Control\CI\Config へ行き、「VulnerableDriverBlocklistEnable」をDWORD値0に設定し、再起動して変更を適用してください。
あるいは、[Microsoftのこちらのページ](https://support.microsoft.com/en-us/topic/kb5020779-the-vulnerable-driver-blocklist-after-the-october-2022-preview-release-3fcbe13a-6013-4118-b584-fcfbc6a09936)を確認してください。

---

## 作成者および貢献者

オリジナル作成者: [https://github.com/z175](https://github.com/z175)

更新および改良: [https://github.com/TheCruZ](https://github.com/TheCruZ)

初期PDBオフセットパーサー作成: [https://github.com/Mohi-eddine](https://github.com/Mohi-eddine)

独立ページ割り当て実装: [https://github.com/Herooyyy/](https://github.com/Herooyyy/)

---

Have Fun!!（楽しんで！）

---
