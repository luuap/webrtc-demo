# webrtc-demo

A simple messaging app using WebRTC. Demonstrates signalling with [luuap/severless-signalling](https://github.com/luuap/serverless-signalling).

## Requirements 
- a running [luuap/severless-signalling](https://github.com/luuap/serverless-signalling) service
- a `.env.local` file with VUE_APP_SIGNALLING_URL

## Installing and running
1. Run `yarn install`
2. Run `yarn dev`
3. In the app ui, click `Create Room`
4. Join the room using the room id (can be used in the same browser, on a different tab)