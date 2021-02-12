---
title: "🐞 Fixing AjaxControlTookit Unexpected Page Reload in ASP.NET Web Form Application"
description: "ASP.NET Web Form Archaeology: What's wrong with my __doPostBack() on Firefox? Why my AjaxControlToolkit is out of Control?"
date: 2021-02-12T12:44:24+08:00
lastmod: 2021-02-12T12:44:24+08:00
draft: false
categories: ["Dev"]
tags: ["pest-control", "aspnet", "webform", "postback", "AjaxControlToolkit", "AsyncPostBackTrigger"]
cover_image: /posts/fixing-ajaxcontroltoolkit-unexpected-page-reload-in-aspnet-webform-application/thumbnail.jpg
cover_image_caption: "Photo by Pixabay from Pexels https://www.pexels.com/photo/adorable-animal-canine-cute-271932/"
---

## Background

{{< toc >}}

TR;DR: see [Summary](#summary) and [the Complete code](#the-complete-code).

In my company there was a old Web Form Website running .NET Framework 3.5. I migrate it to an ASP .NET Web Form Application running .NET Framework 4.7.2 few months ago. The website is running just like the old one but recently a user reported that **a `<select>` list which suppose triggers an ajax call and refresh part of the page does not work. Instead, the page is reloaded every time when the item in the `<select>` list is changed.**

I used almost a day to figure out what's (probably) happened and I hope that this post will be helpgul to someone in the future who (still) encounter this problem.

## Expected vs Actual Behavior

There is a `<form>` contains a `<select>` list created by [ `<asp:DropDownList>`](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.dropdownlist?view=netframework-4.8). When the selected option is changed, an AJAX call will be fired and new items will be fetched. Besides. there is a text area for user to input remarks. A submit button will submit the data in the form.

![image-20210212150215986](img/image-20210212150215986.png)

### Expected

Page will not be reloaded when the selected option is changed. The content in text area will be there.

### Actual

Once the selected option is changed, a form POST is fired (instead of an AJAX POST). The content in text area is therefore being cleaned.

This is related to ajax calls so the first thing I try is searching for how the ajax call is being triggered.

## `AjaxControlToolkit`

In the `.aspx` file I saw a `<asp:UpdatePanel>` and a `<asp:DropDownList>`. This is the simplified structure of that part:

```xml
<asp:UpdatePanel runat="server" ID="UpdatePanel2" UpdateMode="Conditional" ChildrenAsTriggers="true">
  <ContentTemplate>
    <asp:DropDownList runat="server" ID="ddItemTypes" AutoPostBack="true" OnSelectedIndexChanged="ddItemTypes_OnSelectedIndexChanged" />
  </ContentTemplate>            
</asp:UpdatePanel>
```

`AutoPostBack` is there so the app should use `AjaxControlToolkit` to make ajax calls (I guess that's the standard right?).

#### Update Nuget Package

I checked the `package.config`:

```xml
<package id="AjaxControlToolkit" version="4.1.50508" targetFramework="net35" />
```

Looks like the `targetFramework` is not right. And maybe it's time to update the package too.

```xml
<package id="AjaxControlToolkit" version="20.1.0" targetFramework="net472" />
```

#### Migration Guide

[AjaxControlToolkit](https://github.com/DevExpress/AjaxControlToolkit) has some breaking changes since version 15+. In short, you need to:

- Uninstall the old version and install the new version of [`AjaxControlToolkit`](https://www.nuget.org/packages/AjaxControlToolkit).
- Change `<asp:ToolkitScriptManager>` into `<asp:ScriptManager>`
- Remove some unused configs in `web.config` (see [this](https://github.com/DevExpress/AjaxControlToolkit/wiki/Upgrading-from-v7.x-and-below#3---clean-up-webconfig))
- [Optional] Install [`AjaxControlToolkit.HtmlEditor.Sanitizer`](https://www.nuget.org/packages/AjaxControlToolkit.HtmlEditor.Sanitizer/)
  - Change namespace from `AjaxControlToolkit.HTMLEditor` to `AjaxControlToolkit.HtmlEditor`
  - Change namespace from `AjaxControlToolkit.HTMLEditor.ToolbarButton` to `AjaxControlToolkit.HtmlEditor.ToolbarButtons`

[Here](https://github.com/DevExpress/AjaxControlToolkit/wiki/Upgrading-from-v7.x-and-below#3---clean-up-webconfig) is the complete migration guide.

After I upgrade the Nuget package, the ajax partially works. It only works on Chromium-base browsers like Chrome and Edge. However, **it does not work on Firefox (85.0) and Safari (14.0.1)**.

### Different `__doPostBack()` in Firefox

This is what I got from Firefox's dev tool:

![image-20210212140706748](img/image-20210212140706748.png)

The initiator is form `__doPostBack()` in `.aspx` page. In Chrome, it's from `ScriptResource.axd`.

![image-20210212141300293](img/image-20210212141300293.png)

So this is the expected call stack: the call is an XHR fired from `ScriptResource.axd`.

Looks like the items from bottom to `_doPostBack()` are the same, so I dig deeper to `_doPostBack()` and eventually found this piece of code in `ScriptResource.axd`:

```js
if (!this._postBackSettings.async) {            
    form.onsubmit = this._onsubmit;            
    this._originalDoPostBack(eventTarget, eventArgument);            
    form.onsubmit = null;            
    return;        
}
```

if `this._postBackSettings.async` is `false`, `_originaldoPostBack()` will be called and that is the `__doPostBack()` function defined in the `.aspx` page. This will trigger a page reload instead of an AJAX call.

Furthermore, I found that the `asyncTarget` in `_postBackSettings ` is `null` but in Chrome that target is the `<select>` list.

### The Verdict

The javascript event somehow keep propagating to a higher level than the `<select>` list (probably the form level) and therefore being treated as a form POST instead of an AJAX call.

## Adding  `<asp:AsyncPostBackTrigger>`

I did some research related to the Firefox-specific page reload and seems none of them are talking about the issue I met. Therefore I tried to study the fundamentals of this **PostBack** behaviour.

This post [**Avoid (Prevent) Page refresh (PostBack) after SelectedIndexChanged is fired in ASP.Net DropDownList**](https://www.aspsnippets.com/Articles/Avoid-Prevent-Page-refresh-PostBack-after-SelectedIndexChanged-is-fired-in-ASPNet-DropDownList.aspx) shows what I am missing: an `asp:AsyncPostBackTrigger`.

Also, [this StackOverflow answer](https://stackoverflow.com/questions/728043/how-to-stop-updatepanel-from-causing-whole-page-postback/728061#728061) also mention the `asp:AsyncPostBackTrigger` thing. 

**When using the `ScriptManager` (i.e. the one we introduced when updating the `AjaxControlToolkit`), we need to define an `asp:AsyncPostBackTrigger` `Trigger` in order to make the call AJAX.**

Therefore, I try to add a `Trigger`:

```xml
<Triggers>
  <asp:AsyncPostBackTrigger ControlID="ddItemTypes" EventName="SelectedIndexChanged" />
</Triggers>
```

This `Trigger` section should be placed under the same `<asp:UpdatePanel>` with the `<asp:DropDownList>`. Here is the complete snippet:

#### The complete code

```xml
<asp:UpdatePanel runat="server" ID="UpdatePanel2" UpdateMode="Conditional" ChildrenAsTriggers="true">
  <ContentTemplate>
    <asp:DropDownList runat="server" ID="ddItemTypes" AutoPostBack="true" OnSelectedIndexChanged="ddItemTypes_OnSelectedIndexChanged" />
  </ContentTemplate>
  <Triggers>
    <asp:AsyncPostBackTrigger ControlID="ddItemTypes" EventName="SelectedIndexChanged" />
  </Triggers>
</asp:UpdatePanel>
```

`EventName` should be optional but here I want to specify this trigger only listen to the `SelectedIndexChanged` event. The `ControlID` has to be an exact match with the `ID` of `<asp:DropDownList>`.

After I added the `<asp:AsyncPostBackTrigger>`, all the browsers including Chrome, Edge, Firefox (85.0) and Safari (14.0.1) are working.

## Summary

When your .NET 4.5+ ASP.NET Web Form Application using `AjaxControlToolkit` does not work as expected and cause the controls like `<asp:DropDownList>` cannot fire AJAX call (but triggers a form POST), make sure that:

1. The `AjaxControlToolkit` Nuget package is up to date (and did the proper migration steps [here](https://github.com/DevExpress/AjaxControlToolkit/wiki/Upgrading-from-v7.x-and-below#3---clean-up-webconfig)).
2. The `<asp:UpdatePanel>` section should contain a `<Triggers>` section. In that section there is an `<asp:AsyncPostBackTrigger>` with correct `ControlID` same as the `ID` of your control (which is also placed under the same UpdatePanel section).

