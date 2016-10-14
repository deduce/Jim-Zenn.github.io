---
title: Raspberry Pi 采集图像并自动上传
date: 2016-09-08 17:50:24
categories: Code
tags:
- Weekly Project
- Raspeberry Pi
- Timelapse

---

{% fi /uploads/timelapse_1.gif, timelapse, 在 Euclid Hall 屋顶用 Raspberry Pi 拍摄的延时摄影 %}

　　每一次我跑步上 Berkeley Hills，沿途就可以眺望北湾两岸的风景。于是我想要用树莓派在那些观景点完成一个长时间的延时摄影。这周终于付诸实践。
<!-- more -->
　　静态延时摄影说白了就是：将相机固定在一个地点， 过一个固定的时间就拍一张照片，最后在将这些照片组合起来成为一个影片。因为要拍摄大约一天时间，用真的相机或手机去拍，一个怕丢，另一个也要求有这个模式，这个过程也比较费电，人可能也要陪在边上。这两天为了买几个钉子去了一次五金店，看到那里有卖用来做篱笆的木板，忽然有了想法：如果把树莓派、充电宝和相机装在木板上，那么把这个木板随便插在一个有网的地方，那不就可以拍延时摄影了吗。如果 ssh 端口用 ngrok 映射到网络上，还可以远程查看程序的执行情况。拍完的照片上传到谷歌网盘，再删除本地的副本，那么对树莓派的 SD 卡也没有太大的容量要求。
　　总结一下，只需要：Raspberry Pi、Raspberry Pi Camera、充电宝、木板就够了。
　　花4刀买下木板，买了几个对应尺寸的螺丝，一共大概花了6刀。把树莓派和树莓派的相机板钉在木板上，保证不要晃动；再把充电宝固定上去，物理部分就完成了。
　　接下来就是代码。无非是两个部分，一个是调用相机定时拍一张照，这个非常简单。调用 PiCamera 模块两三行就写好了。有意思的是自动上传到 Google Drive。这个部分其实也不复杂，按照 Google 官方的 API 文档大概两个小时就完成了。
　　下附主文件代码(python 2.7.12)。运行时还需要 API Key，这是免费的，直接去谷歌API申请就行。如果哪里有问题，可以留言问我。

``` python

from __future__ import print_function

import time
import datetime
import picamera
import cPickle as pickle

import httplib2
import os

import apiclient
import oauth2client
from oauth2client import client
from oauth2client import tools

import sys

reload(sys)
sys.setdefaultencoding('utf-8')


try:
    import argparse
    flags = argparse.ArgumentParser(parents=[tools.argparser]).parse_args()
except ImportError:
    flags = None

VERSION = '0.1beta'

# OAuth 2.0 scope that will be authorized.
# Check https://developers.google.com/drive/scopes for all available scopes.
SCOPES = 'https://www.googleapis.com/auth/drive'

# Location of the client secrets.
CLIENT_SECRET_FILE = 'client_secrets.json'
APPLICATION_NAME = 'capturePi'

# Folder id
FOLDER_ID = '0B_6TJ0QR4AQLX2lKcDZlQVlDYkU'

# Resolution of the image
RESOLUTION = (1024, 768)

# Where to store the temp data
LOCALPATH = './data/'

# interval
INTERVAL = '180'

def get_credentials():
    """Gets valid user credentials from storage.

    If nothing has been stored, or if the stored credentials are invalid,
    the OAuth2 flow is completed to obtain the new credentials.

    Returns:
    Credentials, the obtained credential.
    """
    home_dir = os.path.expanduser('~')
    credential_dir = os.path.join(home_dir, '.credentials')
    if not os.path.exists(credential_dir):
        os.makedirs(credential_dir)
    credential_path = os.path.join(credential_dir,
            'capturePi_credentials.json')

    store = oauth2client.file.Storage(credential_path)
    credentials = store.get()
    if not credentials or credentials.invalid:
        flow = client.flow_from_clientsecrets(CLIENT_SECRET_FILE, SCOPES)
        flow.user_agent = APPLICATION_NAME
        flow.redirect_uri = oauth2client.client.OOB_CALLBACK_URN
        authorize_url = flow.step1_get_authorize_url()
        print('Go to the following link in your browser: ' + authorize_url)
        code = raw_input('Enter verification code: ').strip()
        credentials = flow.step2_exchange(code)
        store.put(credentials)

        print('Storing credentials to ' + credential_path)
    return credentials

# Create an authorized Drive API client.
credentials = get_credentials()
http = credentials.authorize(httplib2.Http())
service = apiclient.discovery.build('drive', 'v3', http=http)

# Find files
# results = service.files().list(
#     pageSize=100,
#     fields="nextPageToken, files(id, name)",
#     orderBy="starred"
#     ).execute()
# items = results.get('files', [])
# if not items:
#     print('No files found.')
# else:
#     print('Files:')
#     for item in items:
#         print('{0} ({1})'.format(item['name'], item['id']))

print('Connection to Google Drive has been establised.')

# with an instance of PiCamera class
with picamera.PiCamera() as camera:
    # Initializing PiCamera
    camera.resolution = RESOLUTION
    # camera warm-up time
    time.sleep(2)
    # start image capture loop
    while True:
        # Path to the file to be uploaded
        FILENAME = datetime.datetime.now().strftime("%m%d%y%H%M%S") + '.jpg'

        # Metadata about the file.
        MIMETYPE = 'image/jpg'
        TITLE = FILENAME
        DESCRIPTION = 'Caputured by CapturePi ' + VERSION

        # capture the file to ./data
        camera.capture(LOCALPATH + FILENAME)
        print("Captured: ", FILENAME)

        # Upload the image.
        # MediaFileUpload abstracts uploading file contents from a file on disk.
        media_body = apiclient.http.MediaFileUpload(
            LOCALPATH + FILENAME,
            mimetype=MIMETYPE,
            resumable=True
        )

        # The body contains the metadata for the file.
        body = {
        'name': TITLE,
        'title': TITLE,
        'description': DESCRIPTION,
        'parents': [FOLDER_ID]
        }

        # Perform the request and print the result.
        uploaded = 0
        while uploaded == 0:
            try:
                result = service.files().create(
                    body=body,
                    media_body=media_body
                    ).execute()

                if result != 0:
                    uploaded = 1
                    print('Uploaded: ', FILENAME)
                    os.remove(LOCALPATH + FILENAME)
            except:
                uploaded = 0
                print("Failed to submit, try again in 1 minute.")
                sleep(60)

        time.sleep(float(INTERVAL))
```
