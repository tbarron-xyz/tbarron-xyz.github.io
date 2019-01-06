---
layout: post
title: React on both backend and frontend
---

Recently I refactored the express.js app that runs my personal website https://tbarron.xyz/ away from Mustache templates into SSR React. If your server is running in Javascript, why not make its templates Javascript as well? Or, Typescript (TSX).

This required a few steps.

- Creating utility functions `sendComponentAsStaticMarkup` and `sendComponentAsStringAsync` (link)[https://github.com/tbarron-xyz/tbarron.xyz-express-server/blob/a843f26451950c526fb326c4273291ec73902bf1/util/sendComponentAsStaticMarkup.ts]. These accept a React component class (and, optionally, initial props), and return a `(req, res) =>`-style Express route handler, using `ReactDOMServer.renderToStaticMarkup` and `ReactDOMServer.renderToString` respectively.

- In `IndexRouter` which serves up the (landing page)[https://tbarron.xyz/], use `sendComponentAsStaticMarkup(IndexComponent)` in place of `res.render` which uses some template engine that you've configured in Express. This page doesn't use a React frontend, we're just rendering React on the backend, so we use staticMarkup.

- In `TwitchChatStatsRouter` which serves up (the stats app)[https://tbarron.xyz/twitch-chat-monitor], use `sendComponentAsStringAsync(TwitchChatStatsComponent, propsCallback => { .... propsCallback(data); })`. This does have a React frontend, so we want to render to string, not static markup. We are providing a function to provide the initial props for the component as well.

- In `TwitchChatStatsComponent`, the server-side template, we add `<TwitchChatMonitorApp />` (the frontend component) into its container div, and add an "initialState" prop to both `TwitchChatStatsComponent` and `TwitchChatMonitorApp`. I.e., if in the frontend loader you call `ReactDOM.render(TwitchChatMonitorApp, document.getElementById("container"))`, then you would change `<div id="container"></div>` to `<div id="container"><TwitchChatMonitorApp initialState={this.props.initialState} /></div>`.

- Change `ReactDOM.render` to `ReactDOM.hydrate` in the System.js launcher for the frontend code.

- In `TwitchChatMonitorApp` (frontend), in the constructor, set the initial state based on `initialState` if it exists. For me, I already have an method for this, since it's called from within the Websocket onmessage handler.
```
    constructor(props) {
        ...
        if (this.props.initialState) {
            this.updateEmoteByChannel(this.props.initialState.colsToTableData, this.props.initialState.emote);
        }
        ...
    }
        
    updateEmoteByChannel = (newData, emote: string) => {
        this.setState({ colsToTableData: newData, emote: emote });
    }
        ```