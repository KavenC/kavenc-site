---
title: Google Cloud Storage作為Django上傳檔案伺服器
date: 2017-05-27 03:52:08
draft: false
categories:
    - Howto
    - Django
tags:
    - django
    - google cloud platform
    - google cloud storage
description: Django Upload File to Google Cloud Storage
---
## 前言
原本把Django Project放在Heroku免費方案下玩的還滿開心的，直到實作檔案上傳功能時，才發現Heroku Dyno[無法保留上傳的檔案](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem)。只好想辦找檔案伺服器來存放檔案。

稍微查了一下，現在這類服務大概就是Amazon AWS、Google GCP、Microsoft Azure三巨頭在削價競爭。沒搞錯的話，最早成功開啟這個商業模式的應該是AWS，因此網路上大多都推薦使用Amazon S3作為解決方案，並且也有很完整的Django module支援。不過因為AWS的介面我怎樣也用不習慣，所以想試著玩玩看其他兩家平台。其中Google Cloud Storage的價格*應該*是三家中最低的，同時免費方案看起來也比Microsoft更吸引人，所以就決定使用GCP，為地球上最接近天網的公司貢獻更多使用者資料Orz

以下說明將會從申請Google Cloud Storage開始，直到最後接上Django Project，讓使用者可以透過網頁上傳檔案儲存到Google Cloud Storage，並且可以在網頁上下載先前上傳的檔案。

本文適用於**Django 1.11**。

## 申請Google Cloud Storage
首先申請Google Cloud Platform免費試用，目前來說免費試用方案提供12個用300美元的額度。另外大部分的服務都有永久免費額度，只是做個小網站的話基本上是花不到什麼錢的。話雖如此，申請免費試用還是要有信用卡才行。
* [申請免費試用](https://console.cloud.google.com/freetrial)
* [免費方案說明](https://cloud.google.com/free/)

有了免費試用資格後，接著進入[專案資料夾](https://console.cloud.google.com/project)，並建立一個新專案。

![GCP New Project](/images/gcp-new-project.png")

這邊要稍微注意一下，輸入專案名稱並按確定以後，需要大約一分鐘的時間專案才會建立完成，如果出來看到列表還是空的，一分鐘過後再重新整理試試。

專案建立完成以後，點選左方選單Storage進入Google Cloud Storage管理介面。如果左方沒有選單的話，可以按一下左上方的三條線(俗稱Hamburger menu)開啟。

在管理介面中點選**建立Bucket**新增一個Bucket，Google Cloud Storage的儲存空間稱作一個Bucket。

![GCP Class](/images/gcp-class.png")

設定你喜歡的Bucket名稱，選擇Regional Class，並且選擇地區us-west-1。為什麼要這樣選呢，因為這個組合有5GB免費額度。要注意的是，雖然Bucket等級日後也可以更改，但是更改以後只會影響到更改後上傳的檔案，原本在bucket中的檔案還是屬於原本的級別。如果不要求免費的話，建議還是參考一下[每個等級的差異以及收費標準]
(https://cloud.google.com/storage/docs/storage-classes?hl=zh_TW&_ga=1.187665824.922289185.1491723424)，為你的網站慎重選擇一個適合的級別。

Bucket建立好以後就可以透過管理介面上傳下載檔案了。不過如果要讓我在Heroku架設的網站上傳或下載檔案的話，還是要設定一下權限和憑證。

外部網站存取Google Cloud Storage需要先為其申請**服務帳戶(Service Account)**，由左方選單點選IAM與管理->服務帳戶進入管理介面。點選上方建立服務帳戶開始建立帳戶，並輸入以下設定：
- 服務帳戶名稱：取一個自己好辨認的名稱即可
- 角色：設定此帳號的權限，在下拉選單中->儲存空間選單，可以看到儲存空間相關的權限
    - Storage 物件建立者：由於我們將會上傳檔案到儲存空間中，因此需要開啟這個權限
    - Storage 物件檢視者：如果要實作從網站下載檔案，或者是顯示儲存空間中的圖片，則需要開啟這個權限
    - Storage 物件管理員：這個權限包含了上面兩種權限，另外再加上可以為物件修改權限的權限(有點繞口，不過你懂得)，以及刪除物件的權限。一般來說作為網站使用的服務帳戶應該不會需要這個權限。
    - Storage 管理員：可以新增或刪除Bucket的權限，同樣我們也不需要這個權限
    > 有關Storage權限的完整說明可以參考[官方文件](https://cloud.google.com/storage/docs/access-control/iam)
- 提供一組新的私密金鑰
    - 這個JSON檔案將會是我們網站用來認證Storage存取權限的金鑰，服務帳號建立完成後請妥善保存

![GCP Service Account](/images/gcp-service-account.png")

## Django Storage Class 實作
剛開始用`Django Google Cloud Storage`在Google找了一下沒看到現成的module可以使用，大部分都只適用於架設在Google Cloud Platform上的網站。後來才在reddit看到詢問Google Cloud Storage Django module的文章，並且在回覆中有人分享了他的成品：[django-gcloud-storage](https://github.com/strayer/django-gcloud-storage)

這個module實際上已經非常完整合用，不過還是有些小地方需要修改才能符合需求，以下說明如何安裝以及建議的修改：

### 安裝django-gcloud-storage
我們按照作者提供的安裝步驟來設定，首先利用pip安裝django-gcloud-storage module
```bash
pip install django-gcloud-storage
```
接著在settings中增加幾個變數
```python
# 將Django預設檔案存取Class設定使用django_gcloud_storage
DEFAULT_FILE_STORAGE = 'django_gcloud_storage.DjangoGCloudStorage'

# 設定你的Google Cloud Storage Project名稱
GCS_PROJECT = "django-gcloud-storage"

# 設定檔案存放的Bucket名稱
GCS_BUCKET = "django-gcloud-storage-bucket"

# 設定你的金鑰JSON Path
# 必須是是本機的絕對路徑
GCS_CREDENTIALS_FILE_PATH = "/path/to/gcs-credentials.json"
```
這樣就完成了，然後打開網站，嘗試做個檔案上傳，結果發現無法成功上傳....
由錯誤訊息我們可以找到上傳失敗的原因
```
gcloud.exceptions.Forbidden: 403 Caller does not have storage.buckets.get access to bucket
```
訊息顯示由於我們使用的服務帳戶並沒有`storage.bucket.get`的權限，因此被Google Cloud Storage拒絕接受這個請求。問題在於把檔案上傳到Google Cloud Storage，其實並不需要用到storage.bucket.get權限(屬於**Storage管理員**的權限類別)，為什麼我們會發出這個權限請求呢。

追查一下錯誤訊息的backtrace，可以發現這個權限請求來自於`DjangoGCloudStorage`中的這段程式碼
```python
if not self._bucket:
    self._bucket = self.client.get_bucket(self.bucket_name)
```
`self.client.get_bucket()`試圖從伺服器中取回一個Bucket物件，一旦取回這個物件以後，就可以看到Bucket所有相關資訊(比方說權限設定)，因此這個操作必須要有Storage管理員的權限才行。
> 參考Google Cloud Storage Python API [文件](https://googlecloudplatform.github.io/google-cloud-python/stable/storage-client.html)

實際上如果我們只是要從Bucket上傳或下載檔案的話，只需要利用`self.client.bucket()`建立一個本地Bucket物件即可，等到真的要做上傳或下載的操作時，伺服器自然會去驗證這個名稱的Bucket是否存在。因此我們第一個想要修改的地方，就是將取得Bucket物件的方法改成不需要`storage.bucket.get`的權限做法。
> 當然另一個更快解決問題的方法，是將你的服務帳戶加入**Storage管理員**的權限，不過開放沒必要的權限還是讓我感覺不太舒服...

### 修改DjangoGCloudStorage
#### 改用Local Bucket Object
我們可以利用繼承的方式來修改`DjangoGCloudStorage`。先在Project根目錄中建立一個storages目錄，裡面建立`__init__.py`以及`google.py`兩個檔案，並且在`google.py`中寫入下列程式碼
```python
from django_gcloud_storage import DjangoGCloudStorage

class MyGCS(DjangoGCloudStorage):
    @property
    def bucket(self):
        if not self._bucket:
            # 使用 bucket object
            self._bucket = self.client.bucket(self.bucket_name)
        return self._bucket
```
將setting.py中的DEFAULT_STORAGE_CLASS改為我們建立的Class
```python
DEFAULT_FILE_STORAGE = 'storages.google.MyGCS'
```
再試一次上傳即可成功

如果上傳的時候遇到`403 Caller does not have storage.objects.delete access to bucket`，這是因為Bucket中已經有相同名稱的檔案，而覆蓋檔案需要`storage.objects.delete`權限，這個權限必須是**Storage 物件管理員**才會有(我們之前也沒開)。如果上傳檔案的管理是利用Django Model中的ImageField或FileField的話，正常的操作下不會出現覆蓋檔案的行為，因為當Django發現使用者上傳的檔案名稱已經出現在資料庫中時，會自動添加一串亂碼形成一個新的檔案名稱。

若網站真的需要覆蓋檔案的行為，或者需要刪除檔案的行為，只要回到Google Cloud Platform -> IAM服務與管理 -> IAM，為網站使用的服務帳戶增加**Storage 物件管理員**權限即可。

#### 改用Local Blob Object
我們將Bucket改用Local Object操作以後，已避免開放不必要的權限。其實在Blob(也就是Storage中的檔案)操作時，DjangoGCloudStorage也都採用了`get_blob()`從伺服器取得blob object，而實際上有時候我們並不需要取得完整的Blob資訊。雖然使用`get_blob()`並不會遇到權限問題，但是卻可能造成多餘的伺服器操作，間接影響網站載入速度。因此我們可以試著改寫程式碼，盡可能改用`blob()`取得local object來操作。

比較適合改寫的functions有以下三個：
```python
import datetime
from django_gcloud_storage import safe_join, prepare_name, GCloudFile

def _save(self, name, content):
    new_name = safe_join(self.bucket_subdir, name)
    new_name = prepare_name(new_name)

    total_bytes = None if not hasattr(content, 'size') else content.size

    blob = self.bucket.blob(new_name) # 使用 local blob
    blob.upload_from_file(content, size=total_bytes)

    return name

def _open(self, name, mode):
    name = safe_join(self.bucket_subdir, name)
    name = prepare_name(name)

    blob = self.bucket.blob(name) # 使用 local blob
    tmpfile = GCloudFile(blob)
    blob.download_to_file(tmpfile)
    tmpfile.seek(0)

    return tmpfile

def url(self, name):
    name = safe_join(self.bucket_subdir, name)
    name = prepare_name(name)

    # 使用 local blob
    return self.bucket.blob(name).generate_signed_url(
        expiration=datetime.datetime.now() + datetime.timedelta(hours=1))
```
#### 修改金鑰讀取方式
我們現在有一個好用的Google Cloud Storage module了，在local把玩一下覺得都搞定以後，準備發布到網站伺服器時又發現了另一個問題。DjangoGCloudStorage認證方式是透過讀取JSON檔案，實際上Google提供的認證API，也是傳入JSON檔案位置作為參數。對某些Web Hosting來說，這個JSON檔案如何安全的放到Server上，有時候不是那麼好處理的。

比方說我的網站放在Heroku，使用Git上傳，但是把金鑰檔案放在repo中又有安全性問題，何況有些人的Heroku網站是直接跟Github Public Repo連結的，更加不可能將金鑰檔案放在repo裡。

一般金鑰的處理方式都是透過Web Hosting提供設定環境變數的方式，先在Web Hosing Server將金鑰設定在環境變數裡，程式碼中再利用os.environ讀取環境變數來取得金鑰，這樣一來就算公開source code也不會有金鑰外流的危險。於是我們也許可以試著將Google Cloud Platform的金鑰JSON內容字串設定在伺服器環境變數中，接著只要想辦法利用JSON字串來作認證就可以了。

由於先前安裝django-gcloud-storage時也一併安裝了oath2client module，於是翻了一下oath2client source code，發現有一個現成的function可以傳入python dictionary作為service account認證，剛好符合需求。[Reference](https://github.com/google/oauth2client/blob/master/oauth2client/service_account.py#L226)

於是我們著手修改DjangoGCloudStorage的`__init__`以及`client`function如下
```python
import json
import os
from oauth2client.service_account import ServiceAccountCredentials
from django_gcloud_storage import storage
from django.conf import settings

def __init__(self, project=None, bucket=None, credentials=None):
    self._client = None
    self._bucket = None

    if bucket is not None:
        self.bucket_name = bucket
    else:
        self.bucket_name = settings.GCS_BUCKET

    if credentials is not None:
        self.credentials = credentials
    else:
        # 從Django setting中取得參數GCS_CREDENTAILS作為認證依據
        # 這個參數可以是JSON檔案路徑，或者是JSON字串
        self.credentials = settings.GCS_CREDENTIALS

    if project is not None:
        self.project_name = project
    else:
        self.project_name = settings.GCS_PROJECT

    self.bucket_subdir = ''

@property
def client(self):
    if not self._client:
        # 認證的過程中，若認證字串是一個檔案的話
        # 則嘗試用JSON檔案認證的方式
        # 這是原本的認證方式
        if os.path.isfile(self.credentials):
            self._client = storage.Client.from_service_account_json(
                self.credentials, project=self.project_name)
        else:
            # 若認證自傳不是檔案路徑的話，嘗試用json解成dictionary
            # 接著用oauth2client.service_account中的API嘗試認證
            cred_dict = json.loads(self.credentials)
            credentials = ServiceAccountCredentials.from_json_keyfile_dict(
                cred_dict)
            self._client = storage.Client(
                project=self.project_name, credentials=credentials)
    return self._client
```
接著我們將Django Project settings中GCS的認證設定改為
```python
import os
# GCS_CREDENTIALS_FILE_PATH = "/path/to/gcs-credentials.json"
GCS_CREDENTIALS = os.environ['GCS_SERVICE_ACCOUNT']
```
如此一來只要在伺服器上將金鑰JSON的檔案內容設定到下面環境變數`GCS_SERVICE_ACCOUNT`即可。

## 結論
搞Django + GCS的過程中，感覺Google Cloud Platform介面雖然有中文，但是有時候比英文還難懂。並且Google Cloud Platform的文件不知為何，對我來說相當難閱讀，主要是有時候文件分類跟我的邏輯不太吻合，有時候查個資訊總是無法在預期的出現的章節中查到，反而寫在其他看起來不相關的章節。不過還是要說該有的資訊都有，習慣了以後就可以比較快找到正確的位置。

關於DjangoGCloudStorage的改動，我覺得至少local bucket應該是有機會繼進到原作者repo中的，過段時間我再試著發PR過去，這樣以後也不用自己繼承來改了。

以上，祝各位Coding愉快！

## 參考資料
- [Google Cloud Platform Docs](https://cloud.google.com/docs/)
- [Google Cloud Storage Python API Docs](https://googlecloudplatform.github.io/google-cloud-python/stable/storage-client.html)
- [django-gcloud-storage](https://github.com/Strayer/django-gcloud-storage)
