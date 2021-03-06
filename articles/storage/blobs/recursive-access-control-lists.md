---
title: Azure Data Lake Storage Gen2 の ACL を再帰的に設定する | Microsoft Docs
description: 親ディレクトリの既存の子項目に対して、アクセス制御リストを再帰的に追加、更新、および削除することができます。
author: normesta
ms.subservice: data-lake-storage-gen2
ms.service: storage
ms.topic: how-to
ms.date: 01/22/2021
ms.author: normesta
ms.reviewer: prishet
ms.custom: devx-track-csharp, devx-track-azurecli
ms.openlocfilehash: 626e626cbd8fa86bd0366516cbaf5a54789f3988
ms.sourcegitcommit: 6272bc01d8bdb833d43c56375bab1841a9c380a5
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 01/23/2021
ms.locfileid: "98741045"
---
# <a name="set-access-control-lists-acls-recursively-for-azure-data-lake-storage-gen2"></a>Azure Data Lake Storage Gen2 のアクセス制御リスト (ACL) を再帰的に設定する

ACL の継承は、親ディレクトリの下に作成された新しい子項目に対して既に利用可能です。 また、親ディレクトリの既存の子項目に対して ACL を再帰的に追加、更新、および削除することもできます。それぞれの子項目に対してこれらの変更を個別に行う必要はありません。

[ライブラリ](#libraries) | [サンプル](#code-samples) | [ベスト プラクティス](#best-practice-guidelines)

## <a name="prerequisites"></a>前提条件

- Azure サブスクリプション。 [Azure 無料試用版の取得](https://azure.microsoft.com/pricing/free-trial/)に関するページを参照してください。

- 階層型名前空間 (HNS) が有効になっているストレージ アカウント。 作成するには、[こちら](create-data-lake-storage-account.md)の手順に従います。

- 再帰的な ACL プロセスを実行するための適切なアクセス許可。 適切なアクセス許可には、次のいずれかが含まれます。 

  - ターゲット コンテナー、親リソース グループ、またはサブスクリプションのいずれかのスコープで[ストレージ BLOB データ所有者](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner)ロールが割り当てられた、プロビジョニングされた Azure Active Directory (AD) [セキュリティ プリンシパル](../../role-based-access-control/overview.md#security-principal)。   

  - 再帰的な ACL プロセスを適用する予定のターゲット コンテナーまたはディレクトリの所有ユーザー。 これには、ターゲット コンテナーまたはディレクトリ内のすべての子項目が含まれます。 

- ディレクトリとファイルに ACL を適用する方法を理解していること。 「[Azure Data Lake Storage Gen2 のアクセス制御](./data-lake-storage-access-control.md)」を参照してください。 

PowerShell、.NET SDK、および Python SDK のインストール ガイドについては、この記事の「**プロジェクトの設定**」セクションを参照してください。

## <a name="set-up-your-project"></a>プロジェクトの設定

必要なライブラリをインストールします。

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

1. 次のコマンドを使用して、インストールされている PowerShell のバージョンが `5.1` 以降であることを確認します。    

   ```powershell
   echo $PSVersionTable.PSVersion.ToString() 
   ```
    
   お使いの PowerShell のバージョンをアップグレードするには、「[既存の Windows PowerShell をアップグレードする](/powershell/scripting/install/installing-windows-powershell)」を参照してください。
    
2. **Az.Storage** モジュールをインストールします。

   ```powershell
   Install-Module Az.Storage -Repository PSGallery -Force  
   ```

   PowerShell モジュールのインストール方法の詳細については、「[Azure PowerShell モジュールのインストール](/powershell/azure/install-az-ps)」を参照してください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

1. [Azure Cloud Shell](../../cloud-shell/overview.md) を開きます。または、Azure CLI をローカルに[インストール](/cli/azure/install-azure-cli)した場合は、Windows PowerShell などのコマンド コンソール アプリケーションを開きます。

2. 次のコマンドを使用して、インストールされている Azure CLI のバージョンが `2.14.0` 以上であることを確認します。

   ```azurecli
    az --version
   ```
   Azure CLI のバージョンが `2.14.0` より低い場合は、新しいバージョンをインストールします。 「[Azure CLI のインストール](/cli/azure/install-azure-cli)」を参照してください。

### <a name="net"></a>[.NET](#tab/dotnet)

1. コマンド ウィンドウを開きます (例: Windows PowerShell)。

2. プロジェクト ディレクトリから、`dotnet add package` コマンドを使用して Azure.Storage.Files.DataLake プレビュー パッケージをインストールします。

   ```console
   dotnet add package Azure.Storage.Files.DataLake -v 12.5.0-preview.1 -s https://pkgs.dev.azure.com/azure-sdk/public/_packaging/azure-sdk-for-net/nuget/v3/index.json
   ```

3. これらの using ステートメントをコード ファイルの先頭に追加します。

   ```csharp
   using Azure;
   using Azure.Core;
   using Azure.Storage;
   using Azure.Storage.Files.DataLake;
   using Azure.Storage.Files.DataLake.Models;
   using System.Collections.Generic;
   using System.Threading.Tasks;
    ```

### <a name="java"></a>[Java](#tab/java)

開始するには、[こちらのページ](https://search.maven.org/artifact/com.azure/azure-storage-file-datalake)にアクセスして、最新バージョンの Java ライブラリを検索します。 次に、お使いのテキスト エディターで *pom.xml* ファイルを開きます。 該当のバージョンを参照する依存関係要素を追加します。

Azure Active Directory (AD) を使用してクライアント アプリケーションを認証したい場合には、Azure Secret Client Library に依存関係を追加します。 「[プロジェクトに Secret Client Library パッケージを追加する](https://github.com/Azure/azure-sdk-for-java/tree/master/sdk/identity/azure-identity#adding-the-package-to-your-project) 」を参照してください。

次に、以下の import ステートメントをコード ファイルに追加します。

```java
import com.azure.identity.ClientSecretCredential;
import com.azure.identity.ClientSecretCredentialBuilder;
import com.azure.core.http.rest.PagedIterable;
import com.azure.core.http.rest.Response;
import com.azure.storage.common.StorageSharedKeyCredential;
import com.azure.storage.file.datalake.DataLakeDirectoryClient;
import com.azure.storage.file.datalake.DataLakeFileClient;
import com.azure.storage.file.datalake.DataLakeFileSystemClient;
import com.azure.storage.file.datalake.DataLakeServiceClient;
import com.azure.storage.file.datalake.DataLakeServiceClientBuilder;
import com.azure.storage.file.datalake.models.AccessControlChangeResult;
import com.azure.storage.file.datalake.models.AccessControlType;
import com.azure.storage.file.datalake.models.ListPathsOptions;
import com.azure.storage.file.datalake.models.PathAccessControl;
import com.azure.storage.file.datalake.models.PathAccessControlEntry;
import com.azure.storage.file.datalake.models.PathItem;
import com.azure.storage.file.datalake.models.PathPermissions;
import com.azure.storage.file.datalake.models.PathRemoveAccessControlEntry;
import com.azure.storage.file.datalake.models.RolePermissions;
import com.azure.storage.file.datalake.options.PathSetAccessControlRecursiveOptions;
```

### <a name="python"></a>[Python](#tab/python)

1. [Python 向けの Azure Data Lake Storage クライアント ライブラリ](https://recursiveaclpr.blob.core.windows.net/privatedrop/azure_storage_file_datalake-12.1.0b99-py2.py3-none-any.whl?sv=2019-02-02&st=2020-08-24T07%3A47%3A01Z&se=2021-08-25T07%3A47%3A00Z&sr=b&sp=r&sig=H1XYw4FTLJse%2BYQ%2BfamVL21UPVIKRnnh2mfudA%2BfI0I%3D)をダウンロードします。

2. ダウンロードしたライブラリを、[pip](https://pypi.org/project/pip/) を使用してインストールします。

   ```
   pip install azure_storage_file_datalake-12.1.0b99-py2.py3-none-any.whl
   ```

3. 次の import ステートメントを、コード ファイルの先頭に追加します。

   ```python
   import os, uuid, sys
   from azure.storage.filedatalake import DataLakeServiceClient
   from azure.core._match_conditions import MatchConditions
   from azure.storage.filedatalake._models import ContentSettings
   ```

---

## <a name="connect-to-your-account"></a>アカウントに接続する

接続には、Azure Active Directory (AD) を使用するか、アカウント キーを使用することができます。 

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

#### <a name="connect-by-using-azure-active-directory-ad"></a>Azure Active Directory (AD) を使用して接続する

この方法を使用すると、ご利用のユーザー アカウントに、適切な Azure ロールベースのアクセス制御 (Azure RBAC) の割り当てと ACL のアクセス許可がシステムによって確実に付与されます。 

次の表は、サポートされている各ロールとその ACL 設定機能を示しています。

|Role|ACL 設定機能|
|--|--|
|[ストレージ BLOB データ所有者](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner)|アカウント内のすべてのディレクトリとファイル。|
|[ストレージ BLOB データ共同作成者](../../role-based-access-control/built-in-roles.md#storage-blob-data-contributor)|セキュリティ プリンシパルによって所有されているディレクトリとファイルのみ。|

1. Windows PowerShell コマンド ウィンドウを開き、`Connect-AzAccount` コマンドで Azure サブスクリプションにサインインし、画面上の指示に従います。

   ```powershell
   Connect-AzAccount
   ```

2. 自分の ID が複数のサブスクリプションに関連付けられている場合は、アクティブなサブスクリプションを、ディレクトリを作成して管理するストレージ アカウントのサブスクリプションに設定します。 この例では、`<subscription-id>` プレースホルダーの値をサブスクリプションの ID に置き換えます。

   ```powershell
   Select-AzSubscription -SubscriptionId <subscription-id>
   ```
3. ストレージ アカウント コンテキストを取得します。

   ```powershell
   $ctx = New-AzStorageContext -StorageAccountName '<storage-account-name>' -UseConnectedAccount
   ```

#### <a name="connect-by-using-an-account-key"></a>アカウント キーを使用して接続する

この方法を使用する場合、Azure RBAC アクセス許可も、ACL アクセス許可もシステムによってチェックされません。 アカウント キーを使用して、ストレージ アカウント コンテキストを取得します。

```powershell
$ctx = New-AzStorageContext -StorageAccountName "<storage-account-name>" -StorageAccountKey "<storage-account-key>"
```

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

1. Azure CLI をローカルで使用している場合は、login コマンドを実行します。

   ```azurecli
   az login
   ```

   CLI で既定のブラウザーを開くことができる場合、開いたブラウザに Azure サインイン ページが読み込まれます。

   それ以外の場合は、[https://aka.ms/devicelogin](https://aka.ms/devicelogin) でブラウザー ページを開き、ターミナルに表示されている認証コードを入力します。 次に、ブラウザーでアカウントの資格情報を使用してサインインします。

   さまざまな認証方法の詳細については、「[Azure CLI を使用して BLOB またはキュー データへのアクセスを承認する](./authorize-data-operations-cli.md)」を参照してください。

2. 自分の ID が複数のサブスクリプションに関連付けられている場合は、アクティブなサブスクリプションを、静的 Web サイトをホストするストレージ アカウントのサブスクリプションに設定します。

   ```azurecli
   az account set --subscription <subscription-id>
   ```

   `<subscription-id>` プレースホルダーの値をサブスクリプションの ID に置き換えます。

> [!NOTE]
> この記事に示す例は、Azure Active Directory (AD) 認証を示しています。 認証方法の詳細については、「[Azure CLI を使用して BLOB またはキュー データへのアクセスを承認する](./authorize-data-operations-cli.md)」を参照してください。

### <a name="net"></a>[.NET](#tab/dotnet)

この記事のスニペットを使用するには、ストレージ アカウントを表す [DataLakeServiceClient](/dotnet/api/azure.storage.files.datalake.datalakeserviceclient) インスタンスを作成する必要があります。

#### <a name="connect-by-using-azure-active-directory-ad"></a>Azure Active Directory (AD) を使用して接続する

[.NET 用 Azure ID クライアント ライブラリ](https://github.com/Azure/azure-sdk-for-net/tree/master/sdk/identity/Azure.Identity)を使用して、Azure AD でアプリケーションを認証できます。

パッケージをインストールした後、この using ステートメントをコード ファイルの先頭に追加します。

```csharp
using Azure.Identity;
```

クライアント ID、クライアント シークレット、およびテナント ID を取得します。 これを行うには、「[クライアント アプリケーションからの要求を承認するために Azure AD からトークンを取得する](../common/storage-auth-aad-app.md)」を参照してください。 このプロセスの一環として、次のいずれかの[Azure ロールベースのアクセス制御 (Azure RBAC)](../../role-based-access-control/overview.md) ロールをセキュリティ プリンシパルに割り当てる必要があります。 

|Role|ACL 設定機能|
|--|--|
|[ストレージ BLOB データ所有者](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner)|アカウント内のすべてのディレクトリとファイル。|
|[ストレージ BLOB データ共同作成者](../../role-based-access-control/built-in-roles.md#storage-blob-data-contributor)|セキュリティ プリンシパルによって所有されているディレクトリとファイルのみ。|

この例では、クライアント ID、クライアント シークレット、およびテナント ID を使用して [DataLakeServiceClient](/dotnet/api/azure.storage.files.datalake.datalakeserviceclient) インスタンスを作成します。  

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/Authorize_DataLake.cs" id="Snippet_AuthorizeWithAAD":::

#### <a name="connect-by-using-an-account-key"></a>アカウント キーを使用して接続する

このアプローチはアカウントに接続する最も簡単な方法です。 

この例では、アカウント キーを使用して [DataLakeServiceClient](/dotnet/api/azure.storage.files.datalake.datalakeserviceclient) インスタンスを作成します。

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/Authorize_DataLake.cs" id="Snippet_AuthorizeWithKey":::

> [!NOTE]
> その他の例については、[.NET 用 Azure ID クライアント ライブラリ](https://github.com/Azure/azure-sdk-for-net/tree/master/sdk/identity/Azure.Identity)のドキュメントを参照してください。

### <a name="java"></a>[Java](#tab/java)

この記事のスニペットを使用するには、ストレージ アカウントを表す **DataLakeServiceClient** インスタンスを作成する必要があります。 

#### <a name="connect-by-using-an-account-key"></a>アカウント キーを使用して接続する

これはアカウントに接続する最も簡単な方法です。 

この例では、アカウント キーを使用して **DataLakeServiceClient** インスタンスを作成します。

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/Authorize_DataLake.java" id="Snippet_AuthorizeWithKey":::

#### <a name="connect-by-using-azure-active-directory-azure-ad"></a>Azure Active Directory (Azure AD) を使用して接続する

[Java 用 Azure ID クライアント ライブラリ](https://github.com/Azure/azure-sdk-for-java/tree/master/sdk/identity/azure-identity)を使用して、Azure AD でアプリケーションを認証できます。

この例では、クライアント ID、クライアント シークレット、およびテナント ID を使用して、**DataLakeServiceClient** インスタンスを作成します。  これらの値を取得するには、「[クライアント アプリケーションからの要求を承認するために Azure AD からトークンの取得する](../common/storage-auth-aad-app.md)」を参照してください。

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/Authorize_DataLake.java" id="Snippet_AuthorizeWithAzureAD":::

> [!NOTE]
> その他の例については、「[Java 用 Azure ID クライアント ライブラリ](https://github.com/Azure/azure-sdk-for-java/tree/master/sdk/identity/azure-identity)」のドキュメントを参照してください。

### <a name="python"></a>[Python](#tab/python)

この記事のスニペットを使用するには、ストレージ アカウントを表す **DataLakeServiceClient** インスタンスを作成する必要があります。 

### <a name="connect-by-using-azure-active-directory-ad"></a>Azure Active Directory (AD) を使用して接続する

[Python 用 Azure ID クライアント ライブラリ](https://pypi.org/project/azure-identity/)を使用して、Azure AD でアプリケーションを認証できます。

この例では、クライアント ID、クライアント シークレット、およびテナント ID を使用して **DataLakeServiceClient** インスタンスを作成します。  これらの値を取得するには、「[クライアント アプリケーションからの要求を承認するために Azure AD からトークンを取得する](../common/storage-auth-aad-app.md)」を参照してください。 このプロセスの一環として、次のいずれかの[Azure ロールベースのアクセス制御 (Azure RBAC)](../../role-based-access-control/overview.md) ロールをセキュリティ プリンシパルに割り当てる必要があります。 

|Role|ACL 設定機能|
|--|--|
|[ストレージ BLOB データ所有者](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner)|アカウント内のすべてのディレクトリとファイル。|
|[ストレージ BLOB データ共同作成者](../../role-based-access-control/built-in-roles.md#storage-blob-data-contributor)|セキュリティ プリンシパルによって所有されているディレクトリとファイルのみ。|

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/crud_datalake.py" id="Snippet_AuthorizeWithAAD":::

> [!NOTE]
> その他の例については、[Python 用 Azure ID クライアント ライブラリ](https://pypi.org/project/azure-identity/)のドキュメントを参照してください。

### <a name="connect-by-using-an-account-key"></a>アカウント キーを使用して接続する

これはアカウントに接続する最も簡単な方法です。 

この例では、アカウント キーを使用して **DataLakeServiceClient** インスタンスを作成します。

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/crud_datalake.py" id="Snippet_AuthorizeWithKey":::
 
- `storage_account_name` プレースホルダーの値は、実際のストレージ アカウントの名前に置き換えます。

- `storage_account_key` プレースホルダーの値は、実際のストレージ アカウントのアクセス キーに置き換えます。

---

## <a name="set-an-acl-recursively"></a>ACL を再帰的に設定する

ACL を "*設定する*" 場合は、ACL 全体 (そのすべてのエントリを含む) を **置換** します。 セキュリティ プリンシパルのアクセス許可レベルの変更または ACL への新しいセキュリティ プリンシパルの追加を、他の既存のエントリに影響を与えることなく行いたい場合は、代わりに ACL を "*更新*" する必要があります。 ACL を置換するのでなく更新するには、この記事の「[ACL を再帰的に更新する](#update-an-acl-recursively)」セクションを参照してください。  

ACL を "*設定*" する場合は、所有ユーザーのエントリ、所有グループのエントリ、および他のすべてのユーザーのエントリを追加する必要があります。 所有ユーザー、所有グループ、およびその他のすべてのユーザーの詳細については、「[ユーザーと ID](data-lake-storage-access-control.md#users-and-identities)」を参照してください。 

このセクションには、ACL の設定方法の例が含まれています。

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

**Set-AzDataLakeGen2AclRecursive** コマンドレットを使用して、ACL を再帰的に設定します。

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 これらのエントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。

```powershell
$filesystemName = "my-container"
$dirname = "my-parent-directory/"
$userID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";

$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType user -Permission rwx 
$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType group -Permission r-x -InputObject $acl 
$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType other -Permission "---" -InputObject $acl
$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType user -EntityId $userID -Permission r-x -InputObject $acl 

Set-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName -Path $dirname -Acl $acl

```

> [!NOTE]
> **既定の** ACL エントリを設定する場合は、**Set-AzDataLakeGen2ItemAclObject** コマンドを実行するときに **-DefaultScope** パラメーターを使用します。 (例: `$acl = set-AzDataLakeGen2ItemAclObject -AccessControlType user -Permission rwx -DefaultScope`)。

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、[Set-AzDataLakeGen2AclRecursive](/powershell/module/az.storage/set-azdatalakegen2aclrecursive) のリファレンス記事をご覧ください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

[az storage fs access set-recursive](/cli/azure/storage/fs/access#az_storage_fs_access_set_recursive) コマンドを使用して、ACL を再帰的に設定します。

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 これらのエントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。

```azurecli
az storage fs access set-recursive --acl "user::rwx,group::r-x,other::---,user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:r-x" -p my-parent-directory/ -f my-container --account-name mystorageaccount --auth-mode login
```

> [!NOTE]
> **既定** の ACL エントリを設定する場合は、各エントリにプレフィックス `default:` を追加します。 たとえば、`default:user::rwx` または `default:user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:r-x` です。 

### <a name="net"></a>[.NET](#tab/dotnet)

**DataLakeDirectoryClient.SetAccessControlRecursiveAsync** メソッドを呼び出すことによって、ACL を再帰的に設定します。 このメソッドに [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [List](/dotnet/api/system.collections.generic.list-1) を渡します。 [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを設定する場合は、[PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [PathAccessControlItem.DefaultScope](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem.defaultscope#Azure_Storage_Files_DataLake_Models_PathAccessControlItem_DefaultScope) プロパティを **true** に設定できます。 

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 このメソッドは、既定の ACL を設定するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) のコンストラクターで使用されます。 これらの ACL エントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/ACL_DataLake.cs" id="Snippet_SetACLRecursively":::

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、.NET の[サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)をご覧ください。

### <a name="java"></a>[Java](#tab/java)

**DataLakeDirectoryClient.setAccessControlRecursive** メソッドを呼び出すことによって、ACL を再帰的に設定します。 このメソッドに [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) オブジェクトの [List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html) を渡します。 [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを設定する場合は、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドを呼び出し、**true** の値を渡すことができます。 

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 このメソッドは、既定の ACL を設定するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドを呼び出すたびに使用されます。 これらの ACL エントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/ACL_DataLake.java" id="Snippet_SetACLRecursively":::

### <a name="python"></a>[Python](#tab/python)

**DataLakeDirectoryClient.set_access_control_recursive** メソッドを呼び出すことによって、ACL を再帰的に設定します。

**既定の** ACL エントリを設定する場合は、各 ACL エントリ文字列の先頭に文字列 `default:` を追加します。 

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 

このメソッドは、既定の ACL を設定するかどうかを指定する `is_default_scope` という名前のブール型パラメーターを受け取ります。 そのパラメーターが `True` の場合、ACL エントリの一覧の前に文字列 `default:` が付きます。 

これらの ACL エントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。これらのエントリでは、所有ユーザーには読み取り、書き込み、実行のアクセス許可を付与し、所有グループには読み取りと実行のアクセス許可のみを付与し、他のすべてのユーザーにはアクセスを許可しません。 この例の最後の ACL エントリでは、"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" というオブジェクト ID を持つ特定のユーザーに、読み取りと実行のアクセス許可を付与しています。

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/ACL_datalake.py" id="Snippet_SetACLRecursively":::

バッチ サイズを指定してバッチ内の ACL を再帰的に処理する例については、Python の[サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)をご覧ください。

---

## <a name="update-an-acl-recursively"></a>ACL を再帰的に更新する

ACL を "*更新する*" 場合は、ACL を置換するのでなく ACL を変更します。 たとえば、ACL にリストされている他のセキュリティ プリンシパルに影響を与えることなく、新しいセキュリティ プリンシパルを ACL に追加することができます。  ACL を更新するのでなく、置換する場合は、この記事の「[ACL を再帰的に設定する](#set-an-acl-recursively)」セクションを参照してください。 

ACL を更新するには、更新したい ACL エントリを含む新しい ACL オブジェクトを作成してから、そのオブジェクトを ACL の更新操作で使用します。 既存の ACL は取得せずに、更新する ACL エントリを指定するだけです。

このセクションには、ACL の更新方法の例が含まれています。

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

**Update-AzDataLakeGen2AclRecursive** コマンドレットを使用して、ACL を再帰的に更新します。 

この例では、書き込みアクセス許可を持つ ACL エントリを更新します。 

```powershell
$filesystemName = "my-container"
$dirname = "my-parent-directory/"
$userID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";

$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType user -EntityId $userID -Permission rwx

Update-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName -Path $dirname -Acl $acl

```

> [!NOTE]
> **既定の** ACL エントリを更新する場合は、**Set-AzDataLakeGen2ItemAclObject** コマンドを実行するときに **-DefaultScope** パラメーターを使用します。 (例: `$acl = set-AzDataLakeGen2ItemAclObject -AccessControlType user -EntityId $userID -Permission rwx -DefaultScope`)。

バッチ サイズを指定してバッチ内の ACL を再帰的に更新する例については、[Update-AzDataLakeGen2AclRecursive](/powershell/module/az.storage/update-azdatalakegen2aclrecursive) のリファレンス記事をご覧ください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

[az storage fs access update-recursive](/cli/azure/storage/fs/access#az_storage_fs_access_update_recursive) コマンドを使用して、ACL を再帰的に更新します。 

この例では、書き込みアクセス許可を持つ ACL エントリを更新します。 

```azurecli
az storage fs access update-recursive --acl "user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:rwx" -p my-parent-directory/ -f my-container --account-name mystorageaccount --auth-mode login
```

> [!NOTE]
> **既定** の ACL エントリを更新する場合は、各エントリにプレフィックス `default:` を追加します。 たとえば、`default:user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:r-x` のようにします。

### <a name="net"></a>[.NET](#tab/dotnet)

**DataLakeDirectoryClient.UpdateAccessControlRecursiveAsync** メソッドを呼び出すことによって、ACL を再帰的に更新します。  このメソッドに [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [List](/dotnet/api/system.collections.generic.list-1) を渡します。 [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを更新する場合は、[PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [PathAccessControlItem.DefaultScope](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem.defaultscope#Azure_Storage_Files_DataLake_Models_PathAccessControlItem_DefaultScope) プロパティを **true** に設定できます。 

この例では、書き込みアクセス許可を持つ ACL エントリを更新します。 このメソッドは、既定の ACL を更新するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) のコンストラクターで使用されます。

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/ACL_DataLake.cs" id="Snippet_UpdateACLsRecursively":::

バッチ サイズを指定してバッチ内の ACL を再帰的に更新する例については、.NET の[サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)をご覧ください。

### <a name="java"></a>[Java](#tab/java)

**DataLakeDirectoryClient.updateAccessControlRecursive** メソッドを呼び出すことによって、ACL を再帰的に更新します。  このメソッドに [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) オブジェクトの [List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html) を渡します。 [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを更新する場合は、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドを呼び出し、**true** の値を渡すことができます。 

この例では、書き込みアクセス許可を持つ ACL エントリを更新します。 このメソッドは、既定の ACL を更新するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドの呼び出しに使用されます。 

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/ACL_DataLake.java" id="Snippet_UpdateACLRecursively":::

### <a name="python"></a>[Python](#tab/python)

**DataLakeDirectoryClient.update_access_control_recursive** メソッドを呼び出すことによって、ACL を再帰的に更新します。 **既定の** ACL エントリを更新する場合は、各 ACL エントリ文字列の先頭に文字列 `default:` を追加します。 

この例では、書き込みアクセス許可を持つ ACL エントリを更新します。

この例では、`my-parent-directory` という名前のディレクトリの ACL を設定します。 このメソッドは、既定の ACL を更新するかどうかを指定する `is_default_scope` という名前のブール型パラメーターを受け取ります。 そのパラメーターが `True` の場合、更新された ACL エントリの前に文字列 `default:` が付きます。  

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/ACL_datalake.py" id="Snippet_UpdateACLsRecursively":::

バッチ サイズを指定してバッチ内の ACL を再帰的に処理する例については、Python の[サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)をご覧ください。

---

## <a name="remove-acl-entries-recursively"></a>ACL エントリを再帰的に削除する

1 つ以上の ACL エントリを再帰的に削除できます。 ACL エントリを削除するには、削除する ACL エントリ用に新しい ACL オブジェクトを作成してから、そのオブジェクトを ACL の削除操作で使用します。 既存の ACL は取得せずに、削除する ACL エントリを指定するだけです。 

このセクションには、ACL の削除方法の例が含まれています。 

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

**Remove-AzDataLakeGen2AclRecursive** コマンドレットを使用して、ACL エントリを再帰的に削除します。 

この例では、コンテナーのルート ディレクトリから ACL エントリを削除します。  

```powershell
$filesystemName = "my-container"
$userID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

$acl = Set-AzDataLakeGen2ItemAclObject -AccessControlType user -EntityId $userID -Permission "---" 

Remove-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName  -Acl $acl
```

> [!NOTE]
> **既定の** ACL エントリを削除する場合は、**Set-AzDataLakeGen2ItemAclObject** コマンドを実行するときに **-DefaultScope** パラメーターを使用します。 (例: `$acl = set-AzDataLakeGen2ItemAclObject -AccessControlType user -EntityId $userID -Permission "---" -DefaultScope`)。

バッチ サイズを指定してバッチ内の ACL を再帰的に削除する例については、[Remove-AzDataLakeGen2AclRecursive](/powershell/module/az.storage/remove-azdatalakegen2aclrecursive) のリファレンス記事をご覧ください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

[az storage fs access remove-recursive](/cli/azure/storage/fs/access#az_storage_fs_access_remove_recursive) コマンドを使用して、ACL のエントリを削除します。 

この例では、コンテナーのルート ディレクトリから ACL エントリを削除します。  

```azurecli
az storage fs access remove-recursive --acl "user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" -p my-parent-directory/ -f my-container --account-name mystorageaccount --auth-mode login
```

> [!NOTE]
> **既定** の ACL エントリを削除する場合は、各エントリにプレフィックス `default:` を追加します。 たとえば、`default:user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` のようにします。

### <a name="net"></a>[.NET](#tab/dotnet)

**DataLakeDirectoryClient.RemoveAccessControlRecursiveAsync** メソッドを呼び出すことによって、ACL を再帰的に削除します。 このメソッドに [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [List](/dotnet/api/system.collections.generic.list-1) を渡します。 [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを削除する場合は、[PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) の [PathAccessControlItem.DefaultScope](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem.defaultscope#Azure_Storage_Files_DataLake_Models_PathAccessControlItem_DefaultScope) プロパティを **true** に設定できます。 

この例では、`my-parent-directory` という名前のディレクトリの ACL から ACL エントリを削除します。 このメソッドは、既定の ACL からエントリを削除するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは [PathAccessControlItem](/dotnet/api/azure.storage.files.datalake.models.pathaccesscontrolitem) のコンストラクターで使用されます。

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/ACL_DataLake.cs" id="Snippet_RemoveACLRecursively":::

バッチ サイズを指定してバッチ内の ACL を再帰的に削除する例については、.NET の[サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)をご覧ください。

### <a name="java"></a>[Java](#tab/java)

**DataLakeDirectoryClient.removeAccessControlRecursive** メソッドを呼び出すことによって、ACL を再帰的に削除します。 このメソッドに [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) オブジェクトの [List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html) を渡します。 [PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) はそれぞれ ACL エントリを定義します。 

**既定の** ACL エントリを削除する場合は、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドを呼び出し、**true** の値を渡すことができます。  

この例では、`my-parent-directory` という名前のディレクトリの ACL から ACL エントリを削除します。 このメソッドは、既定の ACL からエントリを削除するかどうかを指定する `isDefaultScope` という名前のブール型パラメーターを受け取ります。 そのパラメーターは、[PathAccessControlEntry](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) の **setDefaultScope** メソッドの呼び出しに使用されます。

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/ACL_DataLake.java" id="Snippet_RemoveACLRecursively":::

### <a name="python"></a>[Python](#tab/python)

**DataLakeDirectoryClient.remove_access_control_recursive** メソッドを呼び出すことによって、ACL エントリを削除します。 **既定の** ACL エントリを削除する場合は、ACL エントリ文字列の先頭に文字列 `default:` を追加します。 

この例では、`my-parent-directory` という名前のディレクトリの ACL から ACL エントリを削除します。 このメソッドは、既定の ACL からエントリを削除するかどうかを指定する `is_default_scope` という名前のブール型パラメーターを受け取ります。 そのパラメーターが `True` の場合、更新された ACL エントリの前に文字列 `default:` が付きます。 

```python
def remove_permission_recursively(is_default_scope):
    
    try:
        file_system_client = service_client.get_file_system_client(file_system="my-container")

        directory_client = file_system_client.get_directory_client("my-parent-directory")
              
        acl = 'user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'

        if is_default_scope:
           acl = 'default:user:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'

        directory_client.remove_access_control_recursive(acl=acl)

    except Exception as e:
     print(e)
```

バッチ サイズを指定してバッチ内の ACL を再帰的に処理する例については、Python の[サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)をご覧ください。

---

## <a name="recover-from-failures"></a>エラーからの復旧

実行時エラーまたはアクセス許可エラーが発生する可能性があります。 実行時エラーが発生した場合は、プロセスを最初から再開します。 アクセス許可エラーは、セキュリティ プリンシパルに、変更対象のディレクトリ階層内にあるディレクトリまたはファイルの ACL を変更するための十分なアクセス許可がない場合に発生する可能性があります。 アクセス許可の問題を解決してから、継続トークンを使用してエラーの発生時点からプロセスを再開するか、最初からプロセスを再開するかを選択します。 最初から再開する場合は、継続トークンを使用する必要はありません。 ACL エントリは、悪影響を及ぼすことなく再適用できます。

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

この例では、結果を変数に返し、失敗したエントリを書式設定されたテーブルにパイプします。

```powershell
$result = Set-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName -Path $dirname -Acl $acl
$result
$result.FailedEntries | ft 
```

テーブルの出力に基づいて、アクセス許可のエラーを修正し、継続トークンを使用して実行を再開できます。

```powershell
$result = Set-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName -Path $dirname -Acl $acl -ContinuationToken $result.ContinuationToken
$result

```

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、[Set-AzDataLakeGen2AclRecursive](/powershell/module/az.storage/set-azdatalakegen2aclrecursive) のリファレンス記事をご覧ください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

障害が発生した場合は、`--continue-on-failure` パラメーターを `false` に設定することにより、後続トークンを返すことができます。 エラーに対処した後、コマンドを再度実行し、`--continuation` パラメーターを後続トークンに設定することにより、障害が発生した時点からプロセスを再開できます。 

```azurecli
az storage fs access set-recursive --acl "user::rw-,group::r-x,other::---" --continue-on-failure false --continuation xxxxxxx -p my-parent-directory/ -f my-container --account-name mystorageaccount --auth-mode login  
```

## <a name="net"></a>[.NET](#tab/dotnet)

この例では、エラーが発生した場合に継続トークンを返します。 アプリケーションでは、エラーが解決された後にこの例のメソッドを再び呼び出し、継続トークンを渡すことができます。 この例のメソッドが初めて呼び出される場合は、アプリケーションで継続トークン パラメーターに対して `null` の値を渡すことができます。 

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/ACL_DataLake.cs" id="Snippet_ResumeContinuationToken":::

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、.NET の[サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)をご覧ください。

### <a name="java"></a>[Java](#tab/java)

この例では、エラーが発生した場合に継続トークンを返します。 アプリケーションでは、エラーが解決された後にこの例のメソッドを再び呼び出し、継続トークンを渡すことができます。 この例のメソッドが初めて呼び出される場合は、アプリケーションで継続トークン パラメーターに対して `null` の値を渡すことができます。 

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/ACL_DataLake.java" id="Snippet_ResumeSetACLRecursively":::

### <a name="python"></a>[Python](#tab/python)

この例では、エラーが発生した場合に継続トークンを返します。 アプリケーションでは、エラーが解決された後にこの例のメソッドを再び呼び出し、継続トークンを渡すことができます。 この例のメソッドが初めて呼び出される場合は、アプリケーションで継続トークン パラメーターに対して ``None`` の値を渡すことができます。 

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/ACL_datalake.py" id="Snippet_ResumeContinuationToken":::

バッチ サイズを指定してバッチ内の ACL を再帰的に処理する例については、Python の[サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)をご覧ください。

---

アクセス許可エラーによって中断されずにプロセスを完了したい場合は、それを指定できます。

### <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

この例では、`ContinueOnFailure` パラメーターを使用して、操作中にアクセス許可エラーが検出された場合でも実行が継続されるようにします。 

```powershell
$result = Set-AzDataLakeGen2AclRecursive -Context $ctx -FileSystem $filesystemName -Path $dirname -Acl $acl -ContinueOnFailure

echo "[Result Summary]"
echo "TotalDirectoriesSuccessfulCount: `t$($result.TotalFilesSuccessfulCount)"
echo "TotalFilesSuccessfulCount: `t`t`t$($result.TotalDirectoriesSuccessfulCount)"
echo "TotalFailureCount: `t`t`t`t`t$($result.TotalFailureCount)"
echo "FailedEntries:"$($result.FailedEntries | ft) 
```

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、[Set-AzDataLakeGen2AclRecursive](/powershell/module/az.storage/set-azdatalakegen2aclrecursive) のリファレンス記事をご覧ください。

### <a name="azure-cli"></a>[Azure CLI](#tab/azure-cli)

プロセスが中断されずに確実に完了されるようにするには、`--continue-on-failure` パラメーターを `true` に設定します。 

```azurecli
az storage fs access set-recursive --acl "user::rw-,group::r-x,other::---" --continue-on-failure true --continuation xxxxxxx -p my-parent-directory/ -f my-container --account-name mystorageaccount --auth-mode login  
```

### <a name="net"></a>[.NET](#tab/dotnet)

プロセスが中断されずに確実に完了されるようにするには、**AccessControlChangedOptions** オブジェクトを渡し、そのオブジェクトの **ContinueOnFailure** プロパティを ``true`` に設定します。

この例では、ACL エントリを再帰的に設定します。 このコードでアクセス許可エラーが発生した場合は、そのエラーが記録されて、実行が継続されます。 この例では、エラーの数をコンソールに出力します。 

:::code language="csharp" source="~/azure-storage-snippets/blobs/howto/dotnet/dotnet-v12/ACL_DataLake.cs" id="Snippet_ContinueOnFailure":::

バッチ サイズを指定してバッチ内の ACL を再帰的に設定する例については、.NET の[サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)をご覧ください。

### <a name="java"></a>[Java](#tab/java)

プロセスが中断されずに確実に完了されるようにするには、[PathSetAccessControlRecursiveOptions](https://azuresdkdocs.blob.core.windows.net/$web/java/azure-storage-file-datalake/12.3.0-beta.1/index.html) オブジェクトの **setContinueOnFailure** メソッドを呼び出し、値 **true** を渡します。

この例では、ACL エントリを再帰的に設定します。 このコードでアクセス許可エラーが発生した場合は、そのエラーが記録されて、実行が継続されます。 この例では、エラーの数をコンソールに出力します。 

:::code language="java" source="~/azure-storage-snippets/blobs/howto/Java/Java-v12/src/main/java/com/datalake/manage/ACL_DataLake.java" id="Snippet_ContinueOnFailure":::

### <a name="python"></a>[Python](#tab/python)

プロセスが中断されずに確実に完了されるようにするには、**DataLakeDirectoryClient.set_access_control_recursive** メソッドに後続トークンを渡さないようにします。

この例では、ACL エントリを再帰的に設定します。 このコードでアクセス許可エラーが発生した場合は、そのエラーが記録されて、実行が継続されます。 この例では、エラーの数をコンソールに出力します。 

:::code language="python" source="~/azure-storage-snippets/blobs/howto/python/python-v12/ACL_datalake.py" id="Snippet_ContinueOnFailure":::

バッチ サイズを指定してバッチ内の ACL を再帰的に処理する例については、Python の[サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)をご覧ください。

---

## <a name="resources"></a>リソース

このセクションには、ライブラリとコード サンプルへのリンクが含まれています。

#### <a name="libraries"></a>ライブラリ

- [PowerShell](https://www.powershellgallery.com/packages/Az.Storage/3.0.0)
- [Azure CLI](/cli/azure/storage/fs/access)
- [.NET](https://pkgs.dev.azure.com/azure-sdk/public/_packaging/azure-sdk-for-net/nuget/v3/index.json)
- [Java](/java/api/overview/azure/storage-file-datalake-readme)
- [Python](https://recursiveaclpr.blob.core.windows.net/privatedrop/azure_storage_file_datalake-12.1.0b99-py2.py3-none-any.whl?sv=2019-02-02&st=2020-08-24T07%3A47%3A01Z&se=2021-08-25T07%3A47%3A00Z&sr=b&sp=r&sig=H1XYw4FTLJse%2BYQ%2BfamVL21UPVIKRnnh2mfudA%2BfI0I%3D)
- [REST](/rest/api/storageservices/datalakestoragegen2/path/update)

#### <a name="code-samples"></a>コード サンプル

- PowerShell:[Readme](https://recursiveaclpr.blob.core.windows.net/privatedrop/README.txt?sv=2019-02-02&st=2020-08-24T17%3A03%3A18Z&se=2021-08-25T17%3A03%3A00Z&sr=b&sp=r&sig=sPdKiCSXWExV62sByeOYqBTqpGmV2h9o8BLij3iPkNQ%3D) | [サンプル](https://recursiveaclpr.blob.core.windows.net/privatedrop/samplePS.ps1?sv=2019-02-02&st=2020-08-24T17%3A04%3A44Z&se=2021-08-25T17%3A04%3A00Z&sr=b&sp=r&sig=dNNKS%2BZcp%2F1gl6yOx6QLZ6OpmXkN88ZjBeBtym1Mejo%3D)

- Azure CLI:[サンプル](https://github.com/Azure/azure-cli/blob/2a55a5350696a3a93a13f364f2104ec8bc82cdd3/src/azure-cli/azure/cli/command_modules/storage/docs/ADLS%20Gen2.md)

- NET: [Readme](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FREADME%2520for%2520net%3Fsv%3D2019-02-02%26st%3D2020-08-25T23%253A20%253A42Z%26se%3D2021-08-26T23%253A20%253A00Z%26sr%3Db%26sp%3Dr%26sig%3DKrnHvasHoSoVeUyr2g%2fSc2aDVW3De4A%2fvx0lFWZs494%253D&data=02%7C01%7Cnormesta%40microsoft.com%7Cda902e4fe6c24e6a07d908d8494fd4bd%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637339954503767961&sdata=gd%2B2LphTtDFVb7pZko9rkGO9OG%2FVvmeXprHB9IOEYXE%3D&reserved=0) | [サンプル](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FRecursive-Acl-Sample-Net.zip%3Fsv%3D2019-02-02%26st%3D2020-08-24T07%253A45%253A28Z%26se%3D2021-09-25T07%253A45%253A00Z%26sr%3Db%26sp%3Dr%26sig%3D2GI3f0KaKMZbTi89AgtyGg%252BJePgNSsHKCL68V6I5W3s%253D&data=02%7C01%7Cnormesta%40microsoft.com%7C6eae76c57d224fb6de8908d848525330%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637338865714571853&sdata=%2FWom8iI3DSDMSw%2FfYvAaQ69zbAoqXNTQ39Q9yVMnASA%3D&reserved=0)

- Python: [Readme](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Frecursiveaclpr.blob.core.windows.net%2Fprivatedrop%2FREADME%2520for%2520python%3Fsv%3D2019-02-02%26st%3D2020-08-25T23%253A21%253A47Z%26se%3D2021-08-26T23%253A21%253A00Z%26sr%3Db%26sp%3Dr%26sig%3DRq6Bl5lXrtYk79thy8wX7UTbjyd2f%252B6xzVBFFVYbdYg%253D&data=02%7C01%7Cnormesta%40microsoft.com%7Cda902e4fe6c24e6a07d908d8494fd4bd%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637339954503777915&sdata=3e46Lp2miOHj755Gh0odH3M0%2BdTF3loGCCBENrulVTM%3D&reserved=0) | [サンプル](https://github.com/Azure/azure-sdk-for-python/blob/master/sdk/storage/azure-storage-file-datalake/samples/datalake_samples_access_control_recursive.py)

## <a name="best-practice-guidelines"></a>ベスト プラクティス ガイドライン

このセクションでは、ACL を再帰的に設定するためのベスト プラクティス ガイドラインについて説明します。 

#### <a name="handling-runtime-errors"></a>実行時エラーの処理

実行時エラーは、さまざまな理由で発生する可能性があります (たとえば、停電やクライアント接続の問題など)。 実行時エラーが発生した場合は、再帰的な ACL プロセスを再開します。 ACL は、悪影響を及ぼすことなく項目に再適用できます。 

#### <a name="handling-permission-errors-403"></a>アクセス許可エラー (403) の処理

再帰的な ACL プロセスの実行中にアクセス制御の例外が発生した場合、AD の[セキュリティ プリンシパル](../../role-based-access-control/overview.md#security-principal)に、ディレクトリ階層内にある 1 つ以上の子項目に ACL を適用するための十分なアクセス許可がない可能性があります。 アクセス許可エラーが発生すると、プロセスが停止し、継続トークンが提供されます。 アクセス許可の問題を修正してから、継続トークンを使用して残りのデータセットを処理します。 既に正常に処理済みのディレクトリとファイルをもう一度処理する必要はありません。 再帰的な ACL プロセスを再開することも選択できます。 ACL は、悪影響を及ぼすことなく項目に再適用できます。 

#### <a name="credentials"></a>資格情報 

ターゲット ストレージ アカウントまたはコンテナーのスコープ内の[ストレージ BLOB データ所有者](../../role-based-access-control/built-in-roles.md#storage-blob-data-owner)ロールが割り当てられている Azure AD セキュリティ プリンシパルをプロビジョニングすることをお勧めします。 

#### <a name="performance"></a>パフォーマンス 

待機時間を短縮するには、ストレージ アカウントと同じリージョンにある Azure 仮想マシン (VM) で再帰的な ACL プロセスを実行することをお勧めします。 

#### <a name="acl-limits"></a>ACL の制限

ディレクトリまたはファイルに適用できる ACL の最大数は、アクセス ACL が 32 個と既定 ACL が 32 個です。 詳細については、[Azure Data Lake Storage Gen2 でのアクセス制御](./data-lake-storage-access-control.md)に関するページを参照してください。

## <a name="see-also"></a>関連項目

- [Azure Data Lake Storage Gen2 のアクセス制御](./data-lake-storage-access-control.md)
- [既知の問題](data-lake-storage-known-issues.md)
