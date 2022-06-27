---
title: Automatically detect managed JDKs in Eclipse
date: 2022-06-27
tags: Eclipse, .sdkman, JBang, asdf, Jabba
categories: [Eclipse]
---

I've been using [.sdkman](https://sdkman.io/) to easily install JDKs for a while now, and one thing that bothers me about Eclipse is that it doesn't automatically detect the JDKs that .sdkman manages.

## JRE Discovery
So I wrote the [_**JRE Discovery**_](https://github.com/sidespin/jre-discovery) plugin for Eclipse. It's a simple plugin that automatically detects the JREs that Java managers install, and configures them in Eclipse. It currently detects JREs managed by :

- [SDKMan](https://sdkman.io/),
- [asdf-java](https://github.com/halcyon/asdf-java),
- [Jabba](https://github.com/shyiko/jabba)
- [JBang](https://www.jbang.dev/)

Managed JREs will be automatically discovered on Eclipse startup, or, while running, when added by their respective Java managers.

![Detected JREs](https://github.com/sidespin/jre-discovery/raw/main/images/jre-discovery.png)

Automatic detection can be disabled from the `JRE Discovery` preference page:
![JRE Discovery preference page](https://github.com/sidespin/jre-discovery/raw/main/images/jre-discovery-prefs.png)

## See it in action

Here is an example, when combined with the [JBang / Eclipse Integration](https://marketplace.eclipse.org/content/jbang-eclipse-integration) plugin, you can see the JRE Discovery plugin in action, after declaring the JAVA version to use in your [JBang](https://www.jbang.dev/) script:

![JBang JDK detection](https://user-images.githubusercontent.com/148698/175966754-2e187c51-d2d0-4cf7-93a9-5fe8264156ec.gif)

Pretty awesome or what?

## Installation

_**JRE Discovery**_ is available in the [Eclipse Marketplace](https://marketplace.eclipse.org/content/jre-discovery). Drag the following button to your running Eclipse workspace. (⚠️ *Requires the Eclipse Marketplace Client*)

[![Drag to your running Eclipse* workspace. *Requires Eclipse Marketplace the Client](https://marketplace.eclipse.org/sites/all/themes/solstice/public/images/marketplace/btn-install.svg)](http://marketplace.eclipse.org/marketplace-client-intro?mpc_install=5514555)

Alternatively, in Eclipse:

- open `Help` > `Install New Software...`
- work with: `https://github.com/sidespin/jre-discovery/releases/download/latest/`
- expand the category and select the JRE Discovery Feature
- proceed with the installation
- restart Eclipse

