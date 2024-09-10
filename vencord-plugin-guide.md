# Unofficial Vencord Plugin Guide

# superseded by official docs
**https://docs.vencord.dev**

## Table of contents
  - [Prequisites](#prerequisites)
  - [Setting things up](#setting-things-up)
  - [How a plugin works](#how-a-plugin-works)
  - [Making a plugin](#making-a-plugin)
    - [Writing a patch](#1-adding-the-button)
    - [Writing a component](#2-making-the-button)
    - [Webpack finds](#3-sending-a-message)
    - [Adding settings to a plugin](#4-adding-settings-to-a-plugin)
  - [Additional resources](#additional-resources)
    - [Plugin development](#plugin-development)
    - [Regular expressions](#regular-expressions)

## Prerequisites
  - A [git Vencord install](https://github.com/Vendicated/Vencord/blob/main/docs/1_INSTALLING.md)
  - A fork of Vencord if you plan on making your plugin public
  - A code editor
  - Some very basic knowledge of JavaScript

## Setting things up
First up, you should use a development Vencord build. Assuming you already injected your git install of Vencord (see Prerequisites), you can do it by running a single command
```
pnpm build --watch
```
After that, all you need to do is refresh Discord, and you should see (Dev) next to your Vencord version

![image](https://user-images.githubusercontent.com/78964224/252259960-5b15d7d5-2758-4921-b4ab-833c628b58fa.png)

Now you can start making your plugin. In Vencord's `src` directory, make a `userplugins` folder, and create a TypeScript file named however you'd like. The path should be `[Vencord]/src/userplugins/yourPlugin.ts`

If you want to use multiple files and/or add CSS, you can instead create a folder where the main file is called `index.ts` (`[Vencord]/src/userplugins/yourPlugin/index.ts`)

## How a plugin works
Plugins have a `patches` array, and they are what separate Vencord from other client mods.

Vencord uses a different way of making mods than what you may be used to. Instead of monkeypatching webpack, it directly modifies the code before Discord loads it.

This is *significantly* more efficient than monkeypatching webpack, and is surprisingly easy, but it may be confusing at first. Here's an example:

If you wanted to make a patch that patches the `isStaff` method on a user, the part of code you want to patch would be simmilar to
```js
user.isStaff = function () {
    return someCondition(user);
};
```
Though discord's code is minified, so it's actually closer to
```js
e.isStaff=function(){return n(e)}
```
** **
In settings, you'll notice a new "Test Patch" tab in the Vencord category.

> ℹ If you use VSCode, Vencord has a [companion extension](https://marketplace.visualstudio.com/items?itemName=Vendicated.vencord-companion). It's not required, but can be useful for doing things directly in your code editor

Here's how a patch to make the function always return true would look like
```ts
{ 
    find: ".isStaff=function(){",
    replacement: [{
        match: /\(.isStaff=function\(\){)return \i\(\i\)}/,
        replace: "$1return true;}"
    }]
}
```

- The `find` value is a unique string to find the module you need to patch. It only has to share the module, it doesn't have to be in the exact same function.
In Test Patch, there will be a field to check if a certain finder is unique or not
  - If it's unique, the module number should show at the bottom, which means it's safe to use as a finder. **⚠ Do not rely on minified variable names** like `e` or `n` for your finders - those can change at any time and quickly break your patch
  - If it matches multiple modules, a "Multiple matches. Please refine your filter" error will be shown
  - If it doesn't have any matches, a "No match. Perhaps the module is lazy loaded?" error will be shown. A lazy loaded module is a module only loaded when needed, like the contents of context menus. If you get the error but are sure your patch works, load the code by doing whatever triggers your patch, and then go back to Test Patch

- Within `replacement`, `match` is a regex which matches the relevant part of discord's code that you want to replace, and `replace` is the value to replace your match with.
  - You might've noticed some special groups used in the match, like `$1` and especially `\i`. You can see the meaning of those in the Cheat Sheet present in Test Patch.

I've left a few regex resources in the [Additional resources](#regular-expressions) section

<sub>(todo: add info about [native plugin functions](https://github.com/Vendicated/Vencord/commit/119b628f331e9282342df213f941851f98630ebb))</sub>

## Making a plugin
Let's start off with a template
```ts
import definePlugin from "@utils/types";

export default definePlugin({
    name: "Your Plugin",
    description: "This plugin does something cool",
    authors: [{
        name: "You!",
        id: 0n
    }],

    patches: [],
    start() {

    },
    stop() {

    },
});
```
`name`, `description` and `authors` are self explanatory, `patches` was explained earlier and `start` and `stop` are functions ran when the plugin is started and stopped from settings

> ⚠ As patches require a restart to be applied, plugins that have patches won't be started/stopped until you restart Discord, so the respective `start`/`stop` functions won't be ran instantly. If your plugin doesn't use patches, please remove the `patches` key from the plugin definiton

For this example, let's make a plugin which adds a button to the on click menu of a stock emoji to send that emoji in chat. Not the most useful but it's a good example

![image](https://user-images.githubusercontent.com/78964224/252452559-2d9c19d1-dd28-4fea-a5b7-3d8d862f73da.png)

### 1. Adding the button
When adding something to the UI, the best way to find what component needs to be edited is using React Devtools. Enable "Settings -> Vencord -> Enable React Developer Tools" and fully restart Discord. Now if you open devtools and select the arrow at the top, there should be a new "⚛ Components" option.

In there, you can use the select tool (top left of devtools) to select the element you want to patch. Look for a component that looks useful, and go to the source. The JavaScript when returning a component (after selecting Pretty Print in the bottom left - it makes the code actually readable) looks something like this
```js
return (0, r.jsx)(u.Z, {
  children: [(0, r.jsx)(n.Z, { /* some arguments */ }), (0, x.jsx)(n.Z, { /* some more arguments */ })]
})
```
- The `(0, functionName)(args)` is minifier magic you can ignore - just think of it as `functionName(args)`
- `r.jsx` (or `r.jsxs`) in this context is `React.createElement`
- Since the child of the container is an array, you can add an extra component to it by adding to the array

I'll jump straight for the patch to add the button and then explain it
```ts
{
    find: ".EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION",
    replacement: {
        match: /(?<=.primaryEmoji,src:(\i).{0,400}).Messages.EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION}\)]/,
        replace: "$&.concat([$self.EmojiButton($1)])"
    }
}
```
The finder is the i18n string "EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION". i18n strings and class names (ex. `().emojiName,`) are often the best finders as they are really unlikely to change.

The match makes more sense when you see the original code
```js
/* ... */ q = e => {
let {node: t} = e;
/* ... */
return (0, i.jsxs)(U.Z, {
    className: ee().truncatingText,
    children: [
        (0, l.jsx)(f.default, {
          emojiName: t.name,
          className: Z.primaryEmoji,
          src: t.src
        }),
        /* other children */
        (0, i.jsx)(f.Text, { 
            variant: "text-sm/normal",
            children: z.Z.Messages.EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION
        })
    ]
})
```
First, the patch uses a [lookbehind](https://stackoverflow.com/questions/2973436/regex-lookahead-lookbehind-and-atomic-groups) (`?<=`) to find the variable that stores the emoji, without actually matching code we don't care about and risk accidentally removing it. As the Cheat Sheet mentions, `\i` is a special regex identifier added by Vencord to match variables

> ℹ If you saw any advice about using `arguments` for this before, it's no longer valid ever since the October client mod breakage

Then, after the match (as mentioned earlier, `$&` represents the entire match, so our code is added after it), we `.concat` to the array of components with our own component

### 2. Making the button
Back in your plugin definition, you can add the Button component
```ts
// ...
start() {},
stop() {},
EmojiButton(node) {
    return "placeholder";
},
```
Since we're going to be writing UI with React, you'll likely want to use a TSX file - changing the file extension to `.tsx` will allow you to write React components in your code.

For this, you can (and should, where possible) use Discord's built-in components. Since they're common enoguh, Vencord has it in webpack commons. You can import it at the top of the file, alongside the `definePlugin` import
```ts
import { Button } from "@webpack/common";
```
And you then you can return the button from your function
```tsx
EmojiButton(node) {
   return <Button onClick={() => sendEmote(node)}>
        Send emote
    </Button>;
},
```
However, now we need to make the `sendEmote` function

### 3. Sending a message
The most convinient way to send a message is by using discord's function for it. To find it, you can use webpack finds.

> ℹ Please enable the ConsoleShortcuts plugin, it's really useful.

The main two methods used for finding stuff will be `findByProps` and `findByCode`. In our case, an obvious starting point would be `findByProps("sendMessage")`. Really conveniently, there's only one match, which is the one we care about.

However, there sometimes may be multiple functions which share the same name, even when we only want one. In that case you can pass multiple arguments to the finder, like `findByProps("sendMessage", "editMessage")`. Don't make it too strict though, in case an update breaks it.

Let's add it to the plugin
```ts
import { findByPropsLazy } from "@webpack";
// ...
const { sendMessage } = findByPropsLazy("sendMessage", "editMessage");
```
Wait, why is this one lazy?

Normally, webpack searches cannot be ran at the top level, as it runs before webpack is initalized. By making it lazy, you make the find only run once the value is used, which is after Discord started, and the function is guaranteed to exist. 

Now let's implement the `sendEmote` function
```ts
import { getCurrentChannel } from "@utils/discord";
// ...
interface EmojiNode {
    type: "emoji";
    name: string;
    surrogate: string;
}
function sendEmote(node: EmojiNode) {
    sendMessage(getCurrentChannel().id, {
        content: node.surrogate
    });
}
```
Quite straightforward. We're importing the `getCurrentChannel` function from utils, but other than that there's nothing else to explain.

### 4. Adding settings to a plugin
Let's add a setting that allows you to send the emoji's name alongside the emoji.

You can add settings by importing `definePluginSettings`
```ts
import { definePluginSettings } from "@api/Settings";
import { OptionType } from "@utils/types";
// ...
const settings = definePluginSettings({
    withName: {
        type: OptionType.BOOLEAN,
        description: "Include the emoji's name",
        default: false,
    }
});
```
and then adding it to the plugin object
```ts
export default definePlugin({
    // ...
    settings: settings,
    // or since in JavaScript { value } is treated as { value: value }, you can simply 
    settings,
```
This adds in the settings UI for us. After that, we can add a condition in our `sendEmote` function
```diff
  sendMessage(getCurrentChannel().id, {
-     content: node.surrogate
+     content: settings.store.withName ? `${node.surrogate} - ${node.name.replace(":", "\\:")}` : node.surrogate
  });
```
Settings save automatically, so changes are applied without a restart just fine

Congratulations, we made a basic plugin 🎉

<details>
<summary>Final code</summary>

> If you directly jumped here from the start, take a quick peek back at [writing a component](#2-making-the-button) to change the file extension
```ts
import { definePluginSettings } from "@api/Settings";
import { getCurrentChannel } from "@utils/discord";
import definePlugin, { OptionType } from "@utils/types";
import { findByPropsLazy } from "@webpack";
import { Button } from "@webpack/common";

interface EmojiNode {
    type: "emoji";
    name: string;
    surrogate: string;
}

const { sendMessage } = findByPropsLazy("sendMessage", "editMessage");

function sendEmote(node: EmojiNode) {
    sendMessage(getCurrentChannel().id, {
        content: settings.store.withName ? `${node.surrogate} - ${node.name.replace(":", "\\:")}` : node.surrogate
    });
}

const settings = definePluginSettings({
    withName: {
        type: OptionType.BOOLEAN,
        description: "Include the emoji's name",
        default: false,
    }
});

export default definePlugin({
    name: "Your Plugin",
    description: "This plugin does something cool",
    authors: [{
        name: "You!",
        id: 0n
    }],

    patches: [{
        find: ".EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION",
        replacement: {
            match: /(?<=.primaryEmoji,src:(\i).{0,400}).Messages.EMOJI_POPOUT_STANDARD_EMOJI_DESCRIPTION}\)]/,
            replace: "$&.concat([$self.EmojiButton($1)])"
        }
    }],

    settings,

    EmojiButton(node: EmojiNode) {
        return <Button onClick={() => sendEmote(node)}>
            Send emote
        </Button>;
    }
});
```

</details>

## Additional resources

### Plugin development
- [Discord Webpack CrashCourse](https://gist.github.com/Vendicated/dec16d3fd86f761558ce170d016ebc53)
- [Vencord's CONTRIBUTING.md](https://github.com/Vendicated/Vencord/blob/main/CONTRIBUTING.md)
- [Official Plugin Guide](https://github.com/Vendicated/Vencord/blob/main/docs/2_PLUGINS.md)
- [Unofficial User API Documentation](https://discord-userdoccers.vercel.app/)

### Regular expressions
- [Learn RegEx The Easy Way](https://github.com/ziishaned/learn-regex/blob/master/README.md)
- [iHateRegex](https://ihateregex.io/playground) and [Regex101](https://regex101.com/)
