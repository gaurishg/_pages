---
title: Translango Project
date: 2023-06-12 12:29:00 +0900
categories: [Fullstack, React, FastAPI]
tags: [react, frontend, typescript, react-mui, mui, python, fastapi, aws, gcp]     # TAG names should always be lowercase
---

# Translango - travel without language barriers

<div>
<img style="height:50px; margin: 10px" src="https://upload.wikimedia.org/wikipedia/commons/4/4c/Typescript_logo_2020.svg">
<img style="height:50px; margin: 10px" src="https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg">
<img style="height:50px; margin: 10px" src="https://playwright.dev/java/img/logos/MUI.png">
<img style="height:50px; margin: 10px" src="https://s3.dualstack.us-east-2.amazonaws.com/pythondotorg-assets/media/files/python-logo-only.svg">
<img style="height:50px; margin: 10px" src="https://codeahoy.com/assets/images/compare/python-frameworks/fastapi-logo.png">
<img style="height:50px; margin: 10px" src="https://www.theeggbrussels.com/wp-content/uploads/2018/05/logo-AWS-1024x658.png">
<img style="height:50px; margin: 10px" src="https://www.gend.co/hs-fs/hubfs/gcp-logo-cloud.png?width=730&name=gcp-logo-cloud.png">
</div>



Planning to visit a foreign country but don't know the language? Don't worry, we have got you covered! Translango is a website where you can
- translate text of one language into another
- Upload a picture and get the name of objects in another language
- Play a game to learn new words in a foreign language

It was developed with the help of amazing team of fellow UTokyo students, [Megha Sharma](https://github.com/ms3744), [Yu Nakai](https://github.com/nnaakkaaii), [Rodrigo Curiel](https://github.com/ccurielrodrigo) and [Kunsheng](https://github.com/likunsheng).

We had used following technologies in the project:
- Frontend: React with Typescript + MUI
- Backend: FastAPI with Python
- Cloud Services: Google Cloud Translate for translation; Google Cloud Vision for object detection; Google Cloud Run for backend code deployment and CI/CD for backend, Google Cloud Storage for storing images; AWS Amplify for deploying and setting up CI/CD for frontend
- Docker: We used docker locally as well for deploying our app on the cloud

You can see the site at [https://gaurishg.github.io/TranslanGo/](https://gaurishg.github.io/TranslanGo/).

<img style="max-width: 300px" src="/assets/images/translango/HomePage_en.png">
_Home Page (English)_

<img style="max-width: 300px" src="/assets/images/translango/ChangeLanguages.png">
_Work in your native language_

<img style="max-width: 300px" src="/assets/images/translango/ObjectDetectionAndTranslation.png">
_Detect objects and get translations simultaneously_

<img style="max-width: 300px" src="/assets/images/translango/TextTranslate_en.png">
_Translate Text also_

<img style="max-width: 300px" src="/assets/images/translango/TextTranslate_hi.png">
_Complete App interface in your language_

<img style="max-width: 300px" src="/assets/images/translango/Games.png">
_Gamification_









