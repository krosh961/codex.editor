# CodeX Editor Tools

CodeX Editor is a block-oriented editor. It means that entry composed with the list of `Blocks` of different types: `Texts`, `Headers`, `Images`, `Quotes` etc.

`Tool` — is a class that provide custom `Block` type. All Tools represented by `Plugins`.

Each Tool should have an installation guide.

## Tool class structure

### constructor()

Each Tool's instance called with an params object.

| Param  | Type                | Description                                     |
| ------ | ------------------- | ----------------------------------------------- |
| api    | [`IAPI`][iapi-link] | CodeX Editor's API methods                      |
| config | `object`            | Special configuration params passed in «config» |
| data   | `object`            | Data to be rendered in this Tool                |

[iapi-link]: ../src/types-internal/api.ts

#### Example

```javascript
constructor({data, config, api}) {
  this.data = data;
  this.api = api;
  this.config = config;
  // ...
}
```

### render()

Method that returns Tool's element {HTMLElement} that will be placed into Editor.

### save()

Process Tool's element created by `render()` function in DOM and return Block's data.

### validate(data: BlockToolData): boolean|Promise\<boolean\> _optional_

Allows to check correctness of Tool's data. If data didn't pass the validation it won't be saved. Receives Tool's `data` as input param and returns `boolean` result of validation.

### merge() _optional_

Method that specifies how to merge two `Blocks` of the same type, for example on `Backspace` keypress.
Method does accept data object in same format as the `Render` and it should provide logic how to combine new
data with the currently stored value.

## Internal Tool Settings

Options that Tool can specify. All settings should be passed as static properties of Tool's class.

| Name | Type | Default Value | Description |
| -- | -- | -- | -- |
| `toolbox` | _Object_ | `undefined` | Pass here `icon` and `title` to display this `Tool` in the Editor's `Toolbox` <br /> `icon` - HTML string with icon for Toolbox <br /> `title` - optional title to display in Toolbox |
| `enableLineBreaks` | _Boolean_ | `false` | With this option, CodeX Editor won't handle Enter keydowns. Can be helpful for Tools like `<code>` where line breaks should be handled by default behaviour. |
| `isInline` | _Boolean_ | `false` | Describes Tool as a [Tool for the Inline Toolbar](tools-inline.md) |

## User configuration

All Tools can be configured by users. You can set up some of available settings along with Tool's class
to the `tools` property of Editor Config.

```javascript
var editor = new CodexEditor({
  holderId : 'codex-editor',
  tools: {
    text: {
      class: Text,
      inlineToolbar : true,
      // other settings..
    },
    header: Header
  },
  initialBlock : 'text',
});
```

There are few options available by CodeX Editor.

| Name | Type | Default Value | Description |
| -- | -- | -- | -- |
| `inlineToolbar` | _Boolean/Array_ | `false` | Pass `true` to enable the Inline Toolbar with all Tools, or pass an array with specified Tools list |
| `config` | _Object_ | `null` | User's configuration for Plugin.

## Paste handling

CodeX Editor handles paste on Blocks and provides API for Tools to process the pasted data.

When user pastes content into Editor, pasted content will be splitted into blocks.

1. If plain text will be pasted, it will be splitted by new line characters
2. If HTML string will be pasted, it will be splitted by block tags

Also Editor API allows you to define your own pasting scenario. You can either:

1. Specify **HTML tags**, that can be represented by your Tool. For example, Image Tool can handle `<img>` tags.
If tags you specified will be found on content pasting, your Tool will be rendered.
2. Specify **RegExp** for pasted strings. If pattern has been matched, your Tool will be rendered.
3. Specify **MIME type** or **extensions** of files that can be handled by your Tool on pasting by drag-n-drop or from clipboard.

For each scenario, you should do 2 next things:

1. Define static getter `pasteConfig` in Tool class. Specify handled patterns there.
2. Define public method `onPaste` that will handle PasteEvent to process pasted data.

### HTML tags handling

To handle pasted HTML elements object returned from `pasteConfig` getter should contain following field:

| Name | Type | Description |
| -- | -- | -- |
| `tags` | `String[]` | _Optional_. Should contain all tag names you want to be extracted from pasted data and processed by your `onPaste` method |

For correct work you MUST provide `onPaste` handler at least for `initialBlock` Tool.

> Example

Header Tool can handle `H1`-`H6` tags using paste handling API

```javascript
static get pasteConfig() {
  return {
    tags: ['H1', 'H2', 'H3', 'H4', 'H5', 'H6'],
  }
}
```

> Same tag can be handled by one (first specified) Tool only.

### RegExp patterns handling

Your Tool can analyze text by RegExp patterns to substitute pasted string with data you want. Object returned from `pasteConfig` getter should contain following field to use patterns:

| Name | Type | Description |
| -- | -- | -- |
| `patterns` | `Object` | _Optional_. `patterns` object contains RegExp patterns with their names as object's keys |

**Note** Editor will check pattern's full match, so don't forget to handle all available chars in there.

Pattern will be processed only if paste was on `initialBlock` Tool and pasted string length is less than 450 characters.

> Example

You can handle YouTube links and insert embeded video instead:

```javascript
static get pasteConfig() {
  return {
    patterns: {
      youtube: /http(?:s?):\/\/(?:www\.)?youtu(?:be\.com\/watch\?v=|\.be\/)([\w\-\_]*)(&(amp;)?[\w\?‌​=]*)?/
    },
  }
}
```

### Files pasting

Your Tool can handle files pasted or dropped into the Editor.

To handle file you should provide `files`  property in your `pasteConfig` configuration object.

`files` property is an object with the following fields:

| Name | Type | Description |
| ---- | ---- | ----------- |
| `extensions` | `string[]` | _Optional_ Array of extensions your Tool can handle |
| `mimeTypes` | `sring[]` | _Optional_ Array of MIME types your Tool can handle |

Example

```javascript
static get pasteConfig() {
  return {
    files: {
      mimeTypes: ['image/png'],
      extensions: ['json']
    }
  }
}
```

### Pasted data handling

If you registered some paste substitutions in `pasteConfig` property, you **should** provide `onPaste` callback in your Tool class.
`onPaste` should be public non-static method. It accepts custom _PasteEvent_ object as argument.

PasteEvent is an alias for three types of events - `tag`, `pattern` and `file`. You can get the type from _PasteEvent_ object's `type` property.
Each of these events provide `detail` property with info about pasted content.

| Type  | Detail |
| ----- | ------ |
| `tag` | `data` - pasted HTML element |
| `pattern` | `key` - matched pattern key you specified in `pasteConfig` object <br /> `data` - pasted string |
| `file` | `file` - pasted file |

Example

```javascript
onPaste (event) {
  switch (event.type) {
    case 'tag':
      const element = event.detail.data;

      this.handleHTMLPaste(element);
      break;

    case 'pattern':
      const text = event.detail.data;
      const key = event.detail.key;

      this.handlePatternPaste(key, text);
      break;

    case 'file':
      const file = event.detail.file;

      this.handleFilePaste(file);
      break;
  }
}
```

## Sanitize

CodeX Editor provides [API](sanitizer.md) to clean taint strings.
Use it manually at the `save()` method or or pass `sanitizer` config to do it automatically.

### Sanitizer Configuration

The example of sanitizer configuration

```javascript
let sanitizerConfig = {
  b: true, // leave <b>
  p: true, // leave <p>
}
```

Keys of config object is tags and the values is a rules.

#### Rule

Rule can be boolean, object or function. Object is a dictionary of rules for tag's attributes.

You can set `true`, to allow tag with all attributes or `false|{}` to remove all attributes,
but leave tag.

Also you can pass special attributes that you want to leave.

```javascript
a: {
  href: true
}
```

If you want to use a custom handler, use should specify a function
that returns a rule.

```javascript
b: function(el) {
  return !el.textContent.includes('bad text');
}
```

or

```javascript
a: function(el) {
  let anchorHref = el.getAttribute('href');
  if (anchorHref && anchorHref.substring(0, 4) === 'http') {
    return {
      href: true,
      target: '_blank'
    }
  } else {
    return {
      href: true
    }
  }
}
```

### Manual sanitize

Call API method `sanitizer.clean()` at the save method for each field in returned data.

```javascript
save() {
  return {
    text: this.api.sanitizer.clean(taintString, sanitizerConfig)
  }
}
```

### Automatic sanitize

If you pass the sanitizer config as static getter, CodeX Editor will automatically sanitize your saved data.

Note that if your Tool is allowed to use the Inline Toolbar, we will get sanitizing rules for each Inline Tool
and merge with your passed config.

You can define rules for each field

```javascript
static get sanitize() {
  return {
    text: {},
    items: {
      b: true, // leave <b>
      a: false, // remove <a>
    }
  }
}
```

Don't forget to set the rule for each embedded subitems otherwise they will
not be sanitized.

if you want to sanitize everything and get data without any tags, use `{}` or just
ignore field in case if you want to get pure HTML

```javascript
static get sanitize() {
  return {
    text: {},
    items: {}, // this rules will be used for all properties of this object
    // or
    items: {
      // other objects here won't be sanitized
      subitems: {
        // leave <a> and <b> in subitems
        a: true,
        b: true,
      }
    }
  }
}
```


