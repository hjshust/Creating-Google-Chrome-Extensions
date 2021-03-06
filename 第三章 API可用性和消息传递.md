# 第三章 API可用性和消息传递

本章介绍脚本组件使用Google Chrome扩展框架提供的消息传递API彼此交互的各种方式。 除此之外，您还将了解普通网页(ordinary web pages)如何与扩展程序进行交互。 此外，您还将体验扩展框架提供的许多有用的API用法。 但在所有这一切之前，首先您将了解剩余的输入组件。 像前面的章节一样，本章假设您有一些使用HTML，CSS和JavaScript等技术编写简单网页的经验。 您应该知道网页的事件驱动性质，例如，当点击按钮（使用事件侦听函数等方式）后显示一些UI。除此之外，您还应该了解扩展程序的架构，它是由一些组件组成的，如manifest.json，输入(inputs)，脚本(scripts)和弹出窗口(popups)。 那就让我们开始吧！

### 3.1 输入组件(Input Components)：第二部分

前面的章节省略了一些需要讨论的输入组件。 在开始使用扩展框架中的各种API并了解消息传递API之前，本部分将介绍它们。 这里讨论的输入组件包括多功能输入框(omnibox input)和上下文菜单项(context menu item)。 除了这些组件之外，还讨论了`Content UI`组件的应用。

#### 3.1.1 多功能输入框(omnibox input)

多功能输入框是一个非常特殊的输入组件，可让您使用Google Chrome的地址栏（也称为多功能输入框）注册关键字。 使用此输入组件是非常容易的，因为它只需要一个事件脚本(`event script`)组件和对manifest.json文件一个小的增改，如下所示：

	"omnibox" : {
		"keyword" : "OI"
	}

![](/assets/3-1.png)

如上所示，要使用多功能框输入组件，需要使用`manifest.json`的`omnibox`属性。 此属性具有`keyword`属性。 当相应的keyword属性的值（在这种情况下，OI）（不区分大小写）被输入到地址栏（在得到确认时），用户可以开始与扩展程序进行交互。 如图3-2和图3-3所示。

![](/assets/3-2.png)
![](/assets/3-3.png)

另外，当用户进行交互时，要想在地址栏中使用图标，您可以在manifest.json中定义icons属性（参见代码清单3-1）。 请注意，Chrome会自动创建manifest.json中列出的16像素图标的灰度版本。 另请参见图3-1和3-3，注意图标的彩色和灰色版本之间的区别。

**Listing 3-1.** Chapter 3 /HelloOmniboxInput/ manifest.json

	{
		"manifest_version" : 2,
		"version" : "1.2",
		"name" : "HelloOmniboxInput",
		"description" : "Extension to demonstrate an Omnibox",
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		},
		"omnibox" : {
			"keyword" : "OI"
		},
		"icons" : {
			"16" : "icon-16.png",
			"128" : "icon-128.png"
		}
	}

##### 3.1.1.1 事件脚本在该组件中的角色
现在需要使用这个输入组件的事件脚本部分。 类似于对其他输入组件使用事件脚本，对于多功能框输入组件，事件脚本也用于侦听关联的事件。 这些事件包括`onInputStarted`，`onInputChanged`，`onInputEntered`和`onInputCancelled`。 请注意，这些事件属于chrome.omnibox API。 该API在manifest.json中不需要任何权限。 代码清单3-2中展示了这些事件的使用。

**Listing 3-2.** Chapter 3 /HelloOmniboxInput/ event_script.js

	//region {variables and functions}
	var ON_INPUT_ENTERED_DISPOSITION = {
		"CURRENT_TAB" : "currentTab",
		"NEW_FOREGROUND_TAB" : "newForegroundTab",
		"NEW_BACKGROUND_TAB" : "newBackgroundTab"
	};
	var suggestResultOne = {
		"content" : "Some content",
		"description" : "Description"
	};
	var suggestResults = [suggestResultOne];
	var searchService = "https://www.google.com/";
	searchService += "search?q=chrome+extensions+developers+";
	function CreateWindow(url) {
		var windowCreateData = {"url" : ""};
		windowCreateData.url = url;
		chrome.windows.create(windowCreateData);
	}
	//end-region
	//region {calls}
	chrome.omnibox.onInputStarted.addListener(function() {
		console.log("<InputStarted>");
	});
	chrome.omnibox.onInputChanged.addListener(function(text,suggest) {
		console.log("<InputChanged> Text: " + text);
		//suggest(suggestResults);
		suggest(getSuggestResults(text));
	});
	chrome.omnibox.onInputEntered.addListener(function(text,disposition) {
		console.log("<InputEntered> Text: " + text);
		CreateWindow(searchService + text);
		//default disposition is ON_INPUT_ENTERED_DISPOSITION.CURRENT_TAB
	});
	//end- region

需要监听的最重要的事件是`chrome.omnibox.onInputChanged`和`chrome.omnibox.onInputEntered`。 前一个（事件）的触发意味着用户已经改变了在多功能框中输入的内容，后者（即`onInputEntered`事件）意味着用户已经接受了在多功能框中输入或建议的内容。 与不同事件关联的日志可以在图3-6中看到。

![](/assets/3-4.png)
![](/assets/3-5.png)

当`onInputChanged`事件被触发时，您需要建议结果以供用户接受。 可以通过将数组传递给`suggest`回调来建议结果（参见清单3-2）。 请注意，该数组中的每个元素应该是`SuggestResult`类型，它是一个具有`content` 和 `description`属性的对象。 例如，为了提供一个建议结果`suggestResultOne`，其定义为

	var suggestResultOne = {"content":"Some content","description":"Description"};

您需要将数组`[suggestResultOne]`传递给`suggest`回调。 图3-4和3-5显示了建议的结果，其中建议的结果在图3-4和3-5分别是`Description`和`Search 'tabs' on ... `, `Search 'themes' on ...`。 请注意，实际被接受的是与这些描述相对应的内容。 并且这里“接受”的意思是将`content`传递给`onInputEntered`事件的监听器函数。

![](/assets/3-6.png)

`content`将作为`text`参数传递给此侦听器函数，如代码清单3-2所示。 第二个参数称为`disposition`，是显示结果的推荐上下文。 这通常不是必需的，因为它具有当前选项卡的默认值。 代码清单3-2中的`text`用于在新窗口中显示相关的搜索网页。 请注意，使用`chrome.windows.create`方法创建一个新窗口（参见图3-8）。

![](/assets/3-7.png)

注意如图3-5所示的`HelloOmniboxInput`扩展程序的默认建议`Run HelloOmniboxInput Commond...`。 通过在事件脚本组件中执行以下代码，可以轻松重写在`developer.chrome.com`上搜索（如图3-7所示）：

	chrome.omnibox.setDefaultSuggestion(
		{"description":"Search on developer.chrome.com"}
	);

#### 3.1.2 上下文菜单项(Context Menu Items)

在Chrome扩展框架中，提供的最先进和功能最强大的输入组件之一是上下文菜单项。 该组件允许扩展程序在上下文菜单中创建项目，如图3-10所示。 创建的项目可以嵌套其他此类项目。 当扩展提供多个相关功能（每个显示为单独的上下文菜单项）时，这可以分组在一个项目下，这变得非常有用。 此输入组件需要`contextMenus`权限。

**Listing 3-3.** Chapter 3 /HelloContextMenuItem/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "HelloContextMenuItem",
		"description" : "Extension to demonstrate a Context-Menu-Item",
		"version" : "1.2",
		"permissions" : ["contextMenus"],
		"icons" : {
		"16" : "icon-16.png",
		"128" : "icon-128.png"
		},
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : true
		}
	}

与前一主题中讨论的输入组件类似，使用上下文菜单项组件也非常简单。 首先，您需要在manifest.json中使用`contextMenus`权限。 然后，您需要使用事件脚本组件，以便在上下文菜单中创建项目。 需要在清单中将`persistent`设置为`true`来定义此事件脚本组件，以便在浏览器一打开时它就能够存在。 这是必需的，因为上下文菜单中的项目是从事件脚本组件中定义的，因此它应该持续足够长的时间。 清单3-3提供了事件脚本组件和`contextMenus`权限的相应代码。

**注意：**

>回想一下，persistent属性定义为true的事件脚本是一个后台脚本(`background script`)，如第2章所述。

![](/assets/3-8.png)

##### 3.1.2.1 创建一个菜单

有不同形式的上下文菜单项，包括 `normal`, `separator`, `checkbox` 等。在这里，您将创建一个 `normal` 菜单项。 请注意，如果您的扩展程序需要在上下文菜单中使用多个菜单项，您还可以使用 `separator` 项目（可视地）分组您的菜单项。

**注意：** 

>上下文菜单项可以采取不同的形式。 它可以是 `normal` 菜单（您在上下文菜单中见到的最典型的菜单），`separator`菜单，`checkbox`菜单 或 `radio` 菜单。 对于本书的课程，我们将仅处理前两项。 这将有助于您更多地关注Chrome浏览器中上下文菜单可用的各种上下文。

![](/assets/3-9.png)

每个上下文菜单项都使用唯一的ID进行标识。 在代码清单3-4中，是使用`ID_CONTEXT_MENU_ITEM_HELLO`变量定义的。 接下来是项目可以出现的不同的上下文类型。不同类型的上下文使用清单3-4中的`TYPES_CONTEXT`变量进行定义。 请注意，browser_action和page_action上下文以及弹出组件都与这些按钮的操作相关联。 其余的上下文具有常规含义。 我们不会处理`launcher`上下文。

**Listing 3-4.** Chapter 3 /HelloContextMenuItem/ event_script.js

	//region {variables and functions}
	var consoleGreeting = "Hello World! - from event_script.js";
	function logSuccess(task) {console.log("%s Successful!",task);}
	function logFailure(task) {console.log("%s Failed!",task);}
	var ID_CONTEXT_MENU_ITEM_HELLO = "ID_CONTEXT_MENU_ITEM_HELLO";
	var TYPES_CONTEXT_MENU_ITEM = { //Object used as an enum
		"NORMAL" : "normal",
		"CHECKBOX" : "checkbox",
		"RADIO" : "radio",
		"SEPARATOR" : "separator"
	};
	var TYPES_CONTEXT = {
		"ALL" : "all",
		"PAGE" : "page",
		"FRAME" : "frame",
		"SELECTION" : "selection",
		"LINK" : "link",
		"EDITABLE" : "editable",
		"IMAGE" : "image",
		"VIDEO" : "video",
		"AUDIO" : "audio",
		"LAUNCHER" : "launcher",
		"BROWSER_ACTION" : "browser_action",
		"PAGE_ACTION" : "page_action"
	};
	var match_pattern_stackoverflow = "*://*.stackoverflow.com/*";
	var createProperties = {
		"type" : TYPES_CONTEXT_MENU_ITEM.NORMAL,
		"id" : ID_CONTEXT_MENU_ITEM_HELLO,
		"title" : "Custom search '%s'",
		"contexts" : [TYPES_CONTEXT.SELECTION],
		"documentUrlPatterns" : [match_pattern_stackoverflow],
		//Use "targetUrlPatterns" for TYPES_CONTEXT.IMAGE,
		//TYPES_CONTEXT.VIDEO, TYPES_CONTEXT.AUDIO, etc.
		"targetUrlPatterns" : []
	};
	//end- region

代码清单3-4和3-5中看到的事件脚本代码来自`HelloContextMenuItem`扩展程序。 在此扩展程序中，仅当在访问网页中进行选择时才会创建上下文菜单项（见图3-10）。 另外，为了仅在`stackoverflow.com`主机上使用，也可以在`createProperties`对象中定义`documentUrlPatterns`属性，如清单3-4所示。

![](/assets/3-10.png)


**Listing 3-5.** Chapter3/HelloContextMenuItem/event_script.js

	//region {calls}
	console.log(consoleGreeting);
	chrome.contextMenus.create(createProperties,function() {
		if(!chrome.runtime.lastError) {
			logSuccess("ContextMenus.Create");
			chrome.contextMenus.onClicked.addListener(
				function(info,tab) {
					console.log(
					"id: %s, selection: %s, url: %s",
					info.menuItemId,info.selectionText,
					tab.url
					);
				}
			);
		} else {
			logFailure("ContextMenus.Create");
		}
	});
	//end- region

使用此组件有两个步鄹 - 第一个是创建上下文菜单项（图3-9显示了相应的日志），第二个是在该项目上监听`onClicked`事件。 使用`chrome.contextMenus.create`方法完成上下文菜单项的创建。 此方法将一个对象作为其第一个参数，以创建具有所定义的属性的上下文菜单项。 如果创建不成功，则`chrome.runtime.lastError`对象将捕获最后一个错误。 相反，如果创建成功，您可以为`onClicked`事件定义侦听器函数，如清单代码3-5所示。

![](/assets/3-11.png)

请注意，此侦听器函数接收到一个`info`对象，以及`Tab`对象（它是Tab类型）。 当单击上下文菜单项时，传递的`info`对象包含重要信息。 该信息可用于根据点击的项目采取行动。 例如，在清单3-5中，将`info.menuItemId`属性简单地输出到了控制台（图3-11显示了相应的日志）。 其他重要的属性包括`parentMenuItemId`，`mediaType`，`linkUrl`，`selectionText`等。完整的属性列表可以从URL https://developer.chrome.com/extensions/contextMenus获得。

![](/assets/3-12.png)

#### 3.1.3 重新审视Content-UI

在上一章中，您学习了如何创建一个Content-UI。 但是，除了简单地将其作为网页上的静态内容（参见图2-19）注入，您没有学会如何使用或与之进行交互。 您将在第3章“Exercise Files”文件夹中提供的`HelloContentUI`扩展程序的帮助下练习其使用情况。 在此扩展中，Content-UI用于显示Page-Action组件。

**注意：**

>本书附带的源代码可在http://www.apress.com/9781484217740的“下载”部分获得（免费）。

##### 3.1.3.1 HelloContentUI扩展

此扩展包含一个内容脚本组件，可以从其manifest.json文件中的`matches`属性中推断出，该组件注入到属于example.org主机的受访网页，如清单3-6所示。 除此之外，它还包含一个Page Action和一个事件脚本(event script)组件（参见图3-12）。 请注意，使用`activeTab`权限只能与当前活动的选项卡交互（在本例中，通过内容脚本(`content script`)组件）。

**Listing 3-6.** Chapter 3 /HelloContentUI/manifest.json

	{
		"manifest_version" : 2,
		"name" : "HelloContentUI",
		"description" : "Show Page-Action using Content-UI",
		"version" : "1.2",
		"page_action" : {
			"default_title" : "HelloContentUI",
			"default_icon" : "icon.png",
			"default_popup" : "popup.html"
		},
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		},
		"permissions" : [
			"activeTab"
		],
		"content_scripts" : [
			{
				"matches" : ["*://www.example.org/*"],
				"js" : ["content_script.js"]
			}
		]
	}

![](/assets/3-13.png)

**Listing 3-7.** Chapter 3 /HelloContentUI/content_script.js

	//region {variables and functions}
	var consoleGreeting = "Hello World! - from content_script.js";
	var requestMessage = {"data":"Test message X"};
	var responseCallback = function(responseMessage) {
		console.log("responseMessage: " + responseMessage.data);
	};
	function createButton() {
		var button = document.createElement("button");
		button.style.width = "70px";
		button.style.height = "40px";
		button.style.position = "fixed";
		button.style.top = "10px";
		button.style.right = "10px";
		button.innerText = "Send Message";
		document.body.appendChild(button);
		return button;
	}
	//end-region
	//region {calls}
	console.log(consoleGreeting);
	var button = createButton();
	button.addEventListener("click",function() {
		console.log("Button clicked!");
		chrome.runtime.sendMessage(/*extensionId,*/
			requestMessage,
			responseCallback
		);
	});
	//end- region

如图3-13所示，内容脚本组件（`content_script.js`）用于在其注入的网页中创建一个按钮元素（“Send Message”）。 `addEventListener`方法（参见清单3-7）用于提供一个回调来处理此按钮上的点击事件（回顾第2章，内容脚本组件可以访问所有标准JavaScript API）。点击此按钮后，Page-Action将显示出来。你可能会想知道这是如何工作的。那么这就是消息API变得有用的地方。如上所述，扩展框架中提供的消息传递API允许不同的脚本组件相互交互。这将显示在列表3-7和3-8中。此扩展程序中的内容脚本组件使用消息传递API的`runtime.sendMessage`方法向扩展运行时发送消息。回想一下事件脚本用于表示扩展运行时（请参阅主题“代表运行时脚本”来提醒您自己关于扩展运行时的知识）。

**Listing 3-8.** Chapter 3 /HelloContentUI/ event_script.js

	//region {variables and functions}
	var consoleGreeting = "Hello World! - from event_script.js";
	var responseMessage = {"data":"Test message Y"};
	//end-region
	//region {calls}
	console.log(consoleGreeting);
	//Show Page-Action using the onMessage event
	chrome.runtime.onMessage.addListener(
		function(requestMessage,sender,sendResponse) {
			chrome.pageAction.show(sender.tab.id);
			console.log("requestMessage: " + requestMessage.data);
			sendResponse(responseMessage);
		}
	);
	//end- region

如清单3-8所示，事件脚本组件（event_script.js）可以使用`runtime.onMessage`事件的侦听器函数监听消息。 在这种情况下，监听器函数包含用于显示相应选项卡的Page-Action（见图3-14）的代码，该选项卡是从sender对象的选项卡ID获得的。 不要因为这个例子而感到难以理解，因为本章将会在以下主题中阅读有关消息传递API的更多信息。

![](/assets/3-14.png)

### 3.2 消息通讯

到目前为止，您已经尝试了不同的脚本组件来完成不同的目的。 但是有些情况下，单一类型的脚本组件不足以完成工作，例如，考虑以前讨论的`HelloContentUI`扩展程序。 在该扩展程序中，您需要内容脚本和事件脚本（即扩展运行时）才能相互通信，以显示Page-Action组件。

除此之外，除了属于扩展程序的脚本组件之外，正常（外部）网页也可能想要与扩展程序进行交互。 因此，总共有四种类型的脚本可以彼此通信 - 内容脚本(content scripts)，弹出脚本(popup scripts)，事件脚本(event scripts)(或后台脚本background scripts）和网页脚本(web page scripts, 属于需要与扩展交互的外部网页)。

![](/assets/3-15.png)

#### 3.2.1 API和事件

现在我们可以讨论用于不同脚本组件之间的通信的消息API。 这些API来自标准的JavaScript API和扩展框架。 由标准JavaScript API提供的消息传递API包括`window.postMessage`方法。 而扩展框架提供的消息传递API包括以下方法：

- chrome.runtime.sendMessage
- chrome.runtime.connect (以及相应的 port.postMessage 方法）
- chrome.tabs.sendMessage
- chrome.tabs.connect (以及相应的 port.postMessage 方法)

![](/assets/3-16.png)

标准JavaScript API中与消息传递API相应的事件包括`message`事件。 同样，扩展框架中的消息传递API的相应事件包括以下事件。 现在您已经了解了消息传递的概况，让我们尝试不同的例子来更清楚地了解这些概念。

- chrome.runtime.onMessage
- chrome.runtime.onMessageExternal
- chrome.runtime.onConnect (以及相应的 port.onMessage 事件)
- chrome.runtime.onConnectExternal (以及相应的 port.onMessage 事件)


#### 3.2.2 网页脚本和事件脚本(Web Page Scripts and Event Scripts)
这里的基本思想是允许外部网页与扩展运行时进行通信。 在Chrome浏览器中，所有网页（扩展程序管理页面除外）都可以访问`chrome.runtime.sendMessage`方法，该方法用于将消息发送到扩展运行时。 该方法采用以下参数。

- extensionID
- message
- responseCallback

如清单3-9所示，单击按钮`send_message`将导致`chrome.runtime.sendMessage（extensionID，message，responseCallback）`调用。 请注意，`extensionID`是发送消息的扩展程序的ID。 可以从扩展管理页面查看此ID（参见图3-16）。 另请注意，每次更改扩展程序内容或加载扩展程序文件夹时，此ID都将更改。 此外，对manifest.json（或扩展文件夹中的任何其他文件）的修改只有在重新安装或重新加载扩展程序时才会生效！

**注意：** 

>对于WSandES扩展程序，文件夹WebServer的内容已从本地HTTP服务器提供，以提供接近真实世界场景的演示，在这样的环境里您的扩展可能尝试监听外部网页（在 Chrome浏览器）的消息。另请注意，所提供的网页具有.php后缀，但包含纯HTML代码。 因此，可以在Chrome浏览器中直接打开本地文件网页（即使用file：// scheme）。

![](/assets/3-17.png)

**Listing 3-9.** Chapter 3 /WSandES/WebServer/ webpage_script.js

	//region {variables and functions}
	//Note this from the extensions page
	var extensionID = "lconbphjmfkpdopdnadkdfiiflajajgg";
	var sendMessageButtonID = "send_message";
	var greeting = "Hello World!";
	var message = "Test message X";
	function responseCallback(responseObject) {
		console.log("Message '" +
			responseObject.message + "' from Sender '" +
			responseObject.sender + "'"
		);
	}
	//end-region
	//region {calls}
	console.log(greeting);
	document.addEventListener("DOMContentLoaded",function(dcle) {
		var buttonID = document.getElementById(sendMessageButtonID);
		buttonID.addEventListener("click",function(ce) {
			//This message will be intercepted by event_script.js
			chrome.runtime.sendMessage(extensionID,message,responseCallback);
		});
	});
	//end- region

`sendMessage`中的`message`参数是要发送的消息。 在这种情况下，`message`是字符串“Test message X”。 您可能会对使用的`responseCallback`参数感到疑惑。 那么这个参数是一个回调函数，它被发送消息的接收者，即扩展运行时使用。 对于代码清单3-9和3-10（从WSandES扩展程序）中的示例，事件脚本组件用于表示扩展运行时。 需要注意的一点是，回调（尽管事件脚本调用）在网页脚本的上下文中被执行。 图3-15显示事件脚本中对应于`sendResponse（responseObject）`调用的日志。

![](/assets/3-18.png)

#### 3.2.2.1 监听事件
如代码清单3-10所示，`chrome.runtime.onMessageExternal`事件的侦听器函数用于处理来自外部网页的传入消息。 `onMessageExternal`事件的侦听器函数需要三个参数：`message`，`sender`和`sendResponse`。 如上所述，`sendResponse`是在网页脚本的上下文中执行的回调。 注意传递的参数叫做`responseObject`。

![](/assets/3-19.png)

**Listing 3-10.** Chapter 3 /WSandES/event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var responseObject = {
		message : "Test message Y",
		sender : "event_script.js"
	};
	function GetFormattedMessageString(message,sender) {
		return "Message '" + message + "' from Sender '" + sender.url + "'";
	}
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.runtime.onMessageExternal.addListener(
		function(message,sender,sendResponse) {
			//Will get called from the script where sendResponse is defined
			sendResponse(responseObject);
			console.log(GetFormattedMessageString(message,sender));
		}
	);
	//end- region

`message`参数包含已传递的消息。 同样，sender参数包含发送消息的外部网页的URL。 在这种情况下，外部网页是从本地HTTP服务器提供的网页.php。 对应于这些值的日志如图3-16所示。

##### 3.2.2.2 manifest文件在该API中的角色

对于能够从外部网页（或其他扩展程序）接收消息的扩展，manifest.json文件中需要重要的补充。 这个补充是`external_connectable manifest`属性。 此属性包含两个key——`ids`和`matches`。 这两个键都是数组值。 ids键表示可以发送消息的（外部）扩展程序的ID，而 `matches` 表示可以发送消息的外部网页的URL模式。 需要注意的一个重要的一点是，不管使用的是何种消息传递API（`chrome.runtime.sendMessage`还是`chrome.runtime.connect`），任何需要与外部网页或扩展程序进行通信的扩展程序都必须具有此属性。

![](/assets/3-20.png)

对于WSandES扩展程序（在第3章的“练习文件”文件夹中提供），`external_connectable`属性的定义如下。 这样做只允许从本地主机服务的网页将消息发送到扩展运行时。

	"externally_connectable" : {
		//Extension and app IDs. If this field is not specified, no
		//extensions or apps can communicate.
		//"ids" : [], //To match all extensions and apps, specify only "*"
		//Allowed webpages
		"matches" : ["*://localhost/*"]
	}

##### 3.2.2.3 使用长连接

本节中演示的消息传递示例涉及单次消息和响应。 有时，让对话持续时间比这更长时间是有用的。 为了方便这个目的，扩展框架提供了`chrome.runtime.connect`方法。 请注意，除可以用网页脚本外，此方法也可用于其他脚本，包括内容脚本和弹出脚本。

**注意：** 

>显然，事件脚本也可以使用`chrome.runtime.connect`方法。 但是你很少会发现这样的应用程序。 但是，您一定会在事件脚本中经常使用`chrome.runtime.onConnect`（或`chrome.runtime.onConnectExternal`）事件及其监听器函数。 `connet`方法将主要从事件脚本中使用，以与`另一个扩展程序`进行通信。 对于`sendMessage`方法也是如此。 您可以访问https://developer.chrome.com/extensions/messaging#external，了解有关跨扩展程序消息通信的更多信息。

此方法将`extensionID`作为其第一个参数。 这是您需要长连接到扩展程序的ID。 第二个参数是一个对象，可用于提供有关连接的附加信息，例如{“name”：“connection1”}。 请注意，此方法返回一个`port`对象。 `port.postMessage`方法用于向扩展运行时发送消息。 到目前为止讨论的代码（网页脚本）总结如下。

	var port = chrome.runtime.connect("...",{"name" : "connection1"});
	port.onMessage.addListener(function(message) {
		console.log(message);
	});
	port.postMessage("Test message X");

接下来，在接收端（即使用事件脚本表示的扩展运行时），`chrome.runtime.onConnectExternal`事件的侦听器函数（或用于从扩展中进行连接的`chrome.runtime.onConnect`事件）也接收到 `port `对象。 请注意，在这种情况下 - 即，为了在网页脚本和事件脚本之间进行连接 - 需要使用`onConnectExternal`事件，因为它是在从外部网页（或扩展程序）进行连接时触发。

![](/assets/3-21.png)

最后，使用`port.onMessage`事件的侦听器函数（可用于连接的两端），您可以侦听传入的消息。 请注意，由于每一端都可以访问port对象，所以这意味着它们可以通过建立的连接（通过port）发送以及接收消息。 到目前为止讨论的代码（对于事件脚本）总结如下：

	chrome.runtime.onConnectExternal.addListener(function(port) {
		//if(port.name == "connection1")
		port.onMessage.addListener(function(message) {
			console.log(message); //Test message X
			port.postMessage("Test message Y");
		});
	});

#### 3.2.3 内容脚本和事件脚本

回想一下，内容脚本组件不表示扩展运行时。但是它可以在扩展框架中访问以下API，`extension` ,
`i18n` , `runtime` 以及 `storage` 。而在运行时API中，内容脚本组件可以访问`connect`和`sendMessage`方法。并访问事件`runtime.onConnect`（包括相应的`port.onMessage`事件）和`runtime.onMessage`。

![](/assets/3-22.png)


注意:内容脚本组件无法访问runtime.onMessageExternal和runtime.onConnectExternal事件。 因此，它不能依赖于运行时API与网页脚本进行通信。 相反，它需要监听标准JavaScript API提供的消息事件。首先让我们来看看内容脚本如何与扩展运行时进行通信，然后在以下主题中，您将了解弹出脚本和网页脚本如何与内容脚本通信。

为此，请查看CSandES扩展程序的3-11至3-13代码清单。 从manifest.json文件（清单3-13）注意到，此扩展将内容脚本注入属于本地主机的受访网页。 这是使用匹配数组中的“*：// localhost / *”字符串指定的。 注入的相应脚本是content_script.js（参见图3-17和3-19）。 代码清单3-11包含此脚本的代码。

**注意：**

>对于能够与内容脚本通信的弹出脚本和事件脚本，不能使用`chrome.runtime.sendMessage`方法。 而是需要使用`chrome.tabs.sendMessage`（或`chrome.tabs.connect`）方法。 对于`chrome.tabs.sendMessage`方法，相应的事件保持不变 - 即`chrome.runtime.onMessage`（以及`chrome.tabs.connect`方法的事件`chrome.runtime.onConnect`）。

**Listing 3-11.** Chapter 3 /CSandES/ content_script.js

	//region {variables and functions}
	var sendMessageButtonID = "send_message";
	var greeting = "Hello World!";
	var message = "Test message X";
	function responseCallback(responseObject) {
		console.log("Message '" + responseObject.message +
			"' from Sender '" + responseObject.sender + "'"
		);
	}
	//end-region
	//region {calls}
	console.log(greeting);
	(function(){
		var buttonElement = document.createElement("button");
		buttonElement.style.position = "fixed";
		buttonElement.style.display = "block";
		buttonElement.style.width = "70px";
		buttonElement.style.height = "40px";
		buttonElement.style.bottom = "10px";
		buttonElement.style.left = "10px";
		buttonElement.innerText = "Message Runtime";
		buttonElement.addEventListener("click",function(ce) {
			//This message will be intercepted by event_script.js
			chrome.runtime.sendMessage(message,responseCallback);
		});
		document.body.appendChild(buttonElement);
		/*
			//var port = chrome.runtime.connect("...",{"name":"connection1"});
			var port = chrome.runtime.connect({"name":"connection1"});
			port.onMessage.addListener(function(message){console.log(message);});
			port.postMessage("...");
		*/
	})();
	//end-region

如代码清单3-11所示，一旦内容脚本被注入，它会将一个按钮元素添加到访问的网页中。 此元素具有绑定到其点击事件的侦听器函数。 为了将消息发送到扩展运行时，它调用了`chrome.runtime.sendMessage`方法。 请注意，与以前使用此方法（在代码清单3-9中）相比，此处未使用`extensionID`参数。 这是因为它是一个可选参数。 由于在扩展中正在执行消息传递，因此`extensionID`参数默认为当前扩展的ID。 另请注意，与`sendMessage`方法的使用方式类似，您也可以使用`connect`方法。 再次，`connect`方法的第一个参数也可以省略。

**Listing 3-12.** Chapter 3 /CSandES/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var responseObject = {
		message : "Test message Y",
		sender : "event_script.js"
	};
	function GetFormattedMessageString(message,sender) {
		return "Message '" + message + "' from Sender '" + sender.url + "'";
	}
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.runtime.onMessage.addListener(function(message,sender,sendResponse) {
		//Will get called from the script where sendResponse is defined
		sendResponse(responseObject);
		console.log(GetFormattedMessageString(message,sender));
	});
	/*
	chrome.runtime.onConnect.addListener(function(port) {
		port.onMessage.addListener(function(message){console.log(message);});
		port.postMessage("...");
	});
	*/
	//end-region

![](/assets/3-23.png)

代码清单3-12包含事件脚本的相应代码。 请注意，与上一个示例（参见清单3-10）相比，此处不使用`onMessageExternal`事件。 而是使用`onMessage`事件，因为消息是在在扩展程序中传递。 如果要使用长连接（使用`chrome.runtime.connect`方法;请参见清单3-11），事件脚本的代码将需要`chrome.runtime.onConnect`事件的侦听器函数。 这已经在清单3-12的注释部分显示。 图3-18包含对应于事件脚本的日志。

![](/assets/3-24.png)

**Listing 3-13.** Chapter 3 /CSandES/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "Communication Demo: content-script and event-script",
		"description" : "Shows communication b/w content-script and
		event-script",
		"version" : "1.2",
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		},
		"content_scripts" : [
			{
				"matches" : ["*://localhost/*"],
				"js" : ["content_script.js"]
			}
		]
	}

##### 3.2.3.1 弹出脚本和内容脚本(Popup Scripts and Content Scripts)

现在，我们来看一下弹出脚本如何与内容脚本通信。 为了演示这个，“Exercise Files”文件夹包含`PSandCS`扩展程序。 代码清单3-14至3-16包含此扩展的相关代码。 您可以在浏览器中加载此扩展程序以进行测试。 图3-20和3-21包含分别来自此扩展程序的内容脚本和弹出脚本的日志。

**注意：**
>如果扩展程序的`Browser-Action`（浏览器行为）不包含弹出窗口，则可以使用事件脚本为`chrome.browserAction.onClicked`事件提供侦听器函数。 使用此侦听器函数，可以调用`chrome.tabs.sendMessage`（或`chrome.tabs.connect`）方法。 这种方法不需要使用选项卡API，因为侦听器函数接收活动选项卡作为其参数。 这将仍然需要`activeTab`权限，以便与活动选项卡的内容进行交互，例如，通过`chrome.tabs.executeScript`方法。

从manifest.json文件（代码清单3-14）可以看出，此扩展程序将内容脚本注入属于`example.org`主机的受访网页。 请注意使用`browser_action`属性及其弹出组件。 弹出窗口包含一个可点击的元素（参见图3-21） - “Send Message”，其ID为send_message。 一个监听器函数被附加到此元素上的click事件。 清单3-15包含侦听器函数的相应代码。

**Listing 3-14.** Chapter 3 /PSandCS/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "Communication Demo: popup-script and content-script",
		"description" : "Shows communication b/w popup-script and content-script",
		"version" : "1.2",
		"content_scripts" : [
			{
				"matches" : ["*://www.example.org/*"],
				"js" : ["content_script.js"]
			}
		],
		"browser_action" : {
			"default_title" : "Communication Demo: popup-script and content...",
			"default_icon" : "icon.png",
			"default_popup" : "popup.html"
		}
	}

在清单3-15中，请注意使用`chrome.tabs.query`方法获取（当前）活动选项卡。 如前所述，`chrome.tabs.sendMessage`方法用于向内容脚本发送消息。 该方法采用以下参数。

- tabID
- message
- responseCallback

tabID是发送消息的选项卡的ID。 message参数是要发送的消息。 在这种情况下，它是一个字符串“Test message X”。 并且，类似于之前讨论的扩展程序，`responseCallback`是给发送消息的接收者使用的回调函数。

![](/assets/3-25.png)

**Listing 3-15.** Chapter 3 /PSandCS/ popup_script.js

	//region {variables and functions}
	var sendMessageButtonID = "send_message";
	var greeting = "Hello World!";
	var message = "Test message X";
	function responseCallback(responseObject) {
		console.log("Message '" + responseObject.message +"' from Sender '" + responseObject.sender + "'");
	}
	//end-region
	//region {calls}
	console.log(greeting);
	document.addEventListener("DOMContentLoaded",function(dcle){
		var buttonID = document.getElementById(sendMessageButtonID);
		buttonID.addEventListener("click",function(ce) {
			chrome.tabs.query({"active":true},function(tabs) {
				chrome.tabs.sendMessage(tabs[0].id,message,responseCallback);
			});
		});
	});
	//end- region

最后，内容脚本包含用于接收发送消息的代码。 为此，将使用`chrome.runtime.onMessage`事件的侦听器函数。 在代码清单3-16中，通过调用console.log将收到的消息记录到控制台。 与该日志对应的输出如图3-20所示。 请注意，此扩展中的任何位置都不会使用注入的按钮元素。 您可以通过使用可用的代码来扩展此扩展程序（用于实验等）。

**Listing 3-16.** Chapter 3 /PSandCS/ content_script.js

	//region {variables and functions}
	var consoleGreeting = "Hello World!";
	var responseObject = {
		message : "Test message Y",
		sender : "content_script.js"
	};
	function GetFormattedMessageString(message,sender) {
		return "Message '" + message + "' from Sender '" + sender.id + "'";
	}
	function createButton() {
		var button = document.createElement("button");
		button.style.width = "70px";
		button.style.height = "40px";
		button.style.position = "fixed";
		button.style.top = "10px";
		button.style.right = "10px";
		button.innerText = "Send Message";
		document.body.appendChild(button);
		return button;
	}
	//end-region
	//region {calls}
	console.log(consoleGreeting);
	var button = createButton();
	chrome.runtime.onMessage.addListener(function(message,sender,sendResponse) {
		//Will get called from the script where sendResponse is defined
		sendResponse(responseObject);
		console.log(GetFormattedMessageString(message,sender));
	});
	//end-region

**使用长连接**

要创建一个长连接来与内容脚本进行交互，可以使用`Chrome.tabs.connect`方法。 该方法将选项卡的ID作为其唯一的参数。 在当前扩展程序的指定选项卡中运行的每个内容脚本中触发相应的`chrome.runtime.onConnect`事件。

再次，由于连接的两端都可以访问port对象 - `port.postMessage`方法和相应的`port.onMessage`事件可以用于通信。 这与“网页脚本和事件脚本”章节的方式相似。

![](/assets/3-26.png)

##### 3.2.3.2 内容脚本和网页脚本(Content Scripts and Web Page Scripts)
如前所述，内容脚本无法访问`runtime.onMessageExternal`和`runtime.onConnectExternal`事件。因此，他们不能依赖于运行时API与网页脚本进行通信。相反，他们需要监听标准JavaScript API提供的消息事件。为此，我们来看看本章`Exercise Files`文件夹中提供的`CSandWS`扩展程序。代码3-17至3-19包含此扩展程序的相关代码。

正如在此扩展程序的manifest.json文件中看到的，使用content_scripts属性将内容脚本注入属于本地主机的受访网页。 请注意，为了展示此扩展程序的目的，HTTP服务器不需要服务外部网页。 但是这样做是为了显示安全地使用`window.postMessage`方法和消息事件。 如果内容脚本被注入到本地文件网页中，则postMessage方法中的目标来源（即接收消息的窗口的URL或URI）将需要指定为*。 出于安全原因，这是非常不鼓励的，因为它将允许来自所有主机的消息。

**注意：**
>在此上下文中的本地文件是使用file：// scheme在浏览器中访问的文件。 本地文件网页的一个示例是file：///F：/Exercise%20Files/somefile.html。 这与使用http：//或https：//方案的本地主机提供的网页不同。

**Listing 3-17.** Chapter 3 /CSandWS/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "Communication Demo: content-script and webpage-script",
		"description" : "Shows communication b/w content-script and ...",
		"version" : "1.2",
		"content_scripts" : [
			{
				"matches" : ["*://localhost/*"],
				"js" : ["content_script.js"]
			}
		]
	}

**注意：**
>此扩展程序附带的网页也可以作为本地文件网页。 虽然它具有.php后缀，但它包含纯HTML代码。 因此，它可以在Chrome浏览器中直接打开（即，使用file：//方案作为本地文件网页）。

代码清单3-18包含了注入内容脚本中的代码。 按钮变量是指创建的按钮元素 - Send Message。 这个元素可以在图3-22的右边看到。 请注意，此按钮元素具有与其点击事件相关联的侦听器函数。 单击此元素后，侦听器函数首先将“Button clicked！”字符串记录到控制台。 然后它调用`postMessage 方`法- `window.postMessage（message，targetOrigin）`。 很明显，`message`参数是要发送的消息。 如上所述，`targetOrigin`是将接收消息的窗口的URL或URI。 请注意，在这种情况下，`targetOrigin`指向当前窗口的起始位置：`var targetOrigin = window.location.origin`。 还要注意，`targetOrigin`的所有以下值都无效：“file：//”，“file：// *”，“”和null。 要在本地文件网页中使用消息API，postMessage需要以window.postMessage（消息“*”）方式调用。

**注意：**
>要将内容脚本注入本地文件网页，您需要将matches属性定义为“matches”：[“file：// *”]。

无论提供网页的是哪种方案（即http：//或https：//与file：//），为了接收发送的消息，需要创建消息事件的侦听器函数，如图所示，在`window.addEventListener（“message”，function（me）{/ ** /}`）行中。 代码清单3-19也是这样。 对应于其监听器函数的日志可以在图3-22中看到。 请注意，传递给此监听器函数的event具有data属性，可用于访问由postMessage方法发送的字符串或对象。

![](/assets/3-27.png)

**Listing 3-18.** Chapter 3 /CSandWS/ content_script.js

	//region {variables and functions}
	var consoleGreeting = "Hello World!";
	var targetOrigin = window.location.origin;
	var message = "Test message X";
	function createButton() {
	var button = document.createElement("button");
		button.style.width = "70px";
		button.style.height = "40px";
		button.style.position = "fixed";
		button.style.top = "10px";
		button.style.right = "10px";
		button.innerText = "Send Message";
		document.body.appendChild(button);
		return button;
	}
	//end-region
	//region {calls}
	console.log(consoleGreeting);
	var button = createButton();
	button.addEventListener("click",function() {
		console.log("Button clicked!");
		window.postMessage(message,targetOrigin);
	});
	/*
	window.addEventListener("message",function(me) {
		console.log("message: " + me.data);
	});
	*/
	//end-region

对于此扩展程序，消息传递是从内容脚本发起的。 以类似的方式，您可以扩展此扩展程序以从网页脚本发起消息传递。 您可以在网页脚本中使用现有的监听器函数来执行点击事件。 要处理发送消息，内容脚本应该包含相应的监听器函数。

![](/assets/3-28.png)

**Listing 3-19.** Chapter 3 /CSandWS/WebServer/ webpage_script.js

	//region {variables and functions}
	var sendMessageButtonID = "send_message";
	var greeting = "Hello World!";
	//var targetOrigin = window.location.origin;
	//var message = "Test message Y";
	//end-region
	//region {calls}
	console.log(greeting);
	document.addEventListener("DOMContentLoaded",function(dcle) {
		var buttonID = document.getElementById(sendMessageButtonID);
		buttonID.addEventListener("click",function(ce) {
			//window.postMessage(message,targetOrigin);
		});
	});
	window.addEventListener("message",function(me) {
		console.log("message: " + me.data);
	});
	//end-region

#### 3.2.4 弹出脚本和事件脚本(Popup Scripts and Event Scripts)

现在让我们来看一看弹出脚本与事件脚本是如何通信的。 尽管这两个组件都代表扩展运行时，但它们在扩展架构中作为不同的脚本组件存在。 这就是为什么需要在这两个脚本组件之间进行通信的原因。 使这些组件彼此通信比其他组件容易得多。

**Listing 3-20.** Chapter 3 /PSandES/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "Communication Demo: popup-script and event-script",
		"description" : "Shows communication b/w popup-script and event-script",
		"version" : "1.2",
		"browser_action" : {
			"default_title" : "Communication Demo: popup-script and eventscript",
			"default_icon" : "icon.png",
			"default_popup" : "popup.html"
		},
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		}
	}

由于使用的是弹出脚本，因此很明显扩展程序将需要一个具有弹出窗口的Browser-Action 或 Page-Action组件。 从PSandES扩展程序的代码清单3-20可以看到（在第3章的“Exercise Files”文件夹中），为了使用事件脚本，需要在清单中定义background属性。 代码清单3-22包含相应事件脚本组件的代码。

**Listing 3-21.** Chapter 3 /PSandES/ popup_script.js

	//region {variables and functions}
	var sendMessageButtonID = "send_message";
	var greeting = "Hello World!";
	var message = "Test message X";
	function responseCallback(responseObject) {
		console.log("Message '" + responseObject.message + "' from Sender '" + responseObject.sender + "'");
	}
	//end-region
	//region {calls}
	console.log(greeting);
	document.addEventListener("DOMContentLoaded",function(dcle){
		var buttonID = document.getElementById(sendMessageButtonID);
		buttonID.addEventListener("click",function(ce) {
			//This message will be intercepted by event_script.js
			chrome.runtime.sendMessage(message,responseCallback);
		});
	});
	//end-region

如前面的例子所述，要向事件脚本发送消息，需要进行以下调用：`chrome.runtime.sendMessage（message，response Callback）`。 如你所知，`message`参数是要发送的消息。 `responseCallback`是由发送消息的接收者使用的回调函数。 请注意，清单3-21包含PSandES扩展程序中使用的弹出脚本中的代码。 图3-23和3-25显示了弹出脚本中的日志。

![](/assets/3-29.png)

**Listing 3-22.** Chapter 3 /PSandES/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var responseObject = {
		message : "Test message Y",
		sender : "event_script.js"
	};
	function GetFormattedMessageString(message,sender) {
		return "Message '" + message + "' from Sender '" + sender.url + "'";
	}
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.runtime.onMessage.addListener(function(message,sender,sendResponse) {
		//Will get called from the script where sendResponse is defined
		sendResponse(responseObject);
		console.log(GetFormattedMessageString(message,sender));
	});
	//end-region


最后，为了接收发送的消息，事件脚本为`chrome.runtime.onMessage`事件提供了监听器函数。 对应该函数的日志如图3-24所示。 请注意`sendResponse`回调和传递的参数`responseObject`，它在弹出脚本的上下文中输出到控制台（参见图3-25）。

### 3.3 Google Chrome扩展程序API
Google Chrome扩展程序框架提供了许多特殊用途的API，可让您访问Chrome浏览器的强大功能。 这些API可以访问Chrome浏览器中几乎所有的功能！ 许多API已经被讨论过，尽管它们在不同的上下文中被描述。 例如，以下API被描述为输入组件：`chrome.omnibox`,`chrome.contextMenus`，`chrome.commands`，`chrome.pageAction`和`chrome.browserAction`。

本节将介绍以下API：

- alarms
- bookmarks
- downloads
- history
- notifications
- storage
- tabs

**注意：** 
>扩展程序仍然可以使用浏览器提供给网页的所有标准API（也称为标准JavaScript API）。 这些是您已经熟悉的核心JavaScript和文档对象模型（DOM）API。 此外，还支持XMLHttpRequest（XHR）API，HTML5 API，WebKit API（CSS动画，过滤器等）和V8 API（如JSON）！

除了扩展框架中的这些API之外，本节还介绍了来自标准JavaScript API的XHR API。 其他未在此处讨论的API，但您可以在以下网址获得有关的在线文档：https：//developer.chrome.com/extensions/api_index- 包含以下内容：

- contentSettings
- cookies
- desktopCapture
- extension
- management
- system.cpu
- system.memory
- webstore
- windows

#### 3.3.1 声明权限

要使用大多数`chrome.*` API，您的扩展程序（或应用程序）必须在清单的权限字段中声明其意图。 权限字段中的每个此类权限可以是已知字符串（例如“alarms”，“storage”，“tabs”等）的列表之一 —— 请参阅包含所有权限字符串的以下列表）或匹配模式提供到一个或多个主机访问。 在此之前的所有示例中，您尚未在清单的权限字段中看到匹配模式。 这是因为这样的使用只有在使用到XHR API时才需要（更多关于它的内容本节后面部分讲述）。

##### 3.3.1.1 权限示例

以下代码片段是manifest文件的权限部分的示例。 通过查看此代码你可能已经猜到，需要“alarms”，“tabs”和“bookmarks”权限字符串来分别访问`alarm`，`tab`和`bookmark` API。 与这些权限字符串不同，剩余的两个权限字符串（即URL）需要与扩展程序之外的远程主机进行通信（通过XHR）。

	"permissions" : [
		"alarms", //Extensions-API permission
		"tabs", //Extensions-API permission
		"bookmarks", //Extensions-API permission
		"http://www.blogger.com/", //XHR permission
		"http://*.google.com/" //XHR permission
	],

##### 3.3.1.2 需要权限的API

与Chrome浏览器扩展框架中的API相对应的文档指定了需要一个权限字符串（如果有的话）。 除此之外，如果API要求您在清单中声明一个权限字符串，那么其文档会告诉您如何执行此操作。 例如，在https://developer.chrome.com/extensions/alarms上找到的Alarms API文档显示如何声明`alarm`权限。

- activeTab
- alarms
- audioModem
- background
- bookmarks
- browsingData
- clipboardRead
- clipboardWrite
- contentSettings
- contextMenus
- cookies
- copresence
- debugger
- declarativeContent
- declarativeWebRequest
- desktopCapture
- dns
- documentScan
- downloads
- enterprise.platformKeys
- experimental
- fileBrowserHandler
- fileSystemProvider
- fontSettings
- gcm
- geolocation
- history
- identity
- idle
- idltest
- location
- management
- nativeMessaging
- networking.config
- notificationProvider
- notifications
- pageCapture
- platformKeys
- power
- printerProvider
- privacy
- processes
- proxy
- sessions
- signedInDevices
- storage
- system.cpu
- system.display
- system.memory
- system. storage
- tabCapture
- tabs
- topSites
- tts
- ttsEngine
- unlimitedStorage
- vpnProvider
- wallpaper
- webNavigation
- webRequest
- webRequestBlocking

**注意：** 
>如果您的扩展程序或应用程序受到恶意软件的攻击，权限将有助于减少损失。 在安装扩展（或应用程序）之前，还会向用户显示一些权限，作为警告。 有关这些警告的更多信息，请访问https://developer.chrome.com/extensions/permission_warnings。

##### 3.3.1.3 可选权限

有两种类型的权限 - 必需的权限和可选权限。目前为止你所处理的权限都是必需的权限。 可选权限是可以在运行时请求的权限，而不是安装时。 用户了解为什么需要这个权限，并且仅授予必要的权限。 虽然可选权限对用户来说更具信息，但由于其复杂性，因此不会在此描述。 您可以访问https://developer.chrome.com/extensions/permissions了解更多相关信息。

#### 3.3.2 Alarm API
Alarm API（即chrome.alarms API）用于使代码定期执行或在指定的时间运行。 它使用alarm权限。 清单3-23包含Alarms API扩展程序的代码，在第3章的“Exercise Files”文件夹中提供。

![](/assets/3-30.png)

要创建Alarm，请使用`chrome.alarms.create`方法。 这个方法需要两个参数 - `alarmName`（String类型）和`alarmInfo`（Object类型）。 `alarmInfo`对象描述闹钟何时触发。 初始时间必须由`when`或`delayInMinutes`指定（但不能同时指定）。

![](/assets/3-31.png)

如果设置了`periodInMinutes`，则在事件初始化后的periodInMinutes内，alarm将周期性地重复。 当一个alarm已经过去时，相应的`chrome.alarms.onAlarm`事件的侦听器功能被触发。 图3-26显示来自此监听器函数的日志。 注意此回调所接收的参数。 此参数的类型为Alarm，其中包含`name`，`scheduledTime`和`periodInMinutes`等属性。

![](/assets/3-32.png)

**注意：**
>在调试扩展程序（已解压缩）的情况下，alarm可以触发的频率没有限制。 对于所有其他情况，间隔不到一分钟的警报将不会被支持，并会引起警告。 请参阅主题“加载扩展文件夹”（从第1章），提醒自己加载扩展程序的基础知识。

**Listing 3-23.** Chapter 3 /AlarmsAPI/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var count = 0;
	var alarmName = "testAlarm";
	var alarmInfo = {
		when : Date.now() + 6000,
		periodInMinutes : 1 //Repeatedly fire after every 1 minute
	};
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.alarms.clearAll();
	chrome.alarms.onAlarm.addListener(function(alarm) {
		console.log("onAlarm-" + ++count);
	});
	chrome.alarms.create(alarmName,alarmInfo);
	//end-region

在清单3-23中，使用`chrome.alarms.clearAll`方法清除所有`alarm`。 要清除特定的`alarm`，请使用`chrome.alarms.clear（string name, function callback）`方法。 alarms API中的其他重要方法包括以下内容。 这里注意，`get`方法的回调函数接收一个类型为Alarm的参数，而`getAll`方法的回调函数接收一个数组类型的参数。

- chrome.alarms.get(string name, function callback
- chrome.alarms.getAll(function callback)

#### 3.3.3 Bookmarks API

Bookmarks API（即chrome.bookmarks API）用于创建，组织和以其他方式操作书签。 它使用书签权限。 书签以树形结构组织，树中的每个节点都是书签或文件夹。 树中的每个节点都由一个`bookmarks.BookmarkTreeNode`对象表示。 `BookmarkTreeNode`属性的使用遍布chrome.bookmarks API。 例如，当您调用`chrome.bookmarks.create`时，您传递新节点的父（`parentId`），以及可选的节点的`title`和`URL`属性。 要了解节点可以拥有的完整属性列表，请参阅https://developer.chrome.com/extensions/bookmarks#type-BookmarkTreeNode。

![](/assets/3-33.png)

请注意，如果一个节点是一个文件夹，它具有以下属性`id`，`parentId`，`children`和`title`。 如果它是一个书签，它具有以下属性`id`，`parentId`，`title`和`url`。 此外，书签树的根节点没有任何父节点，因此没有任何`parentId`。 它有以下两个特殊文件夹作为其孩子 - 书签栏和其他书签（Bookmarks Bar and Other Bookmarks）。

**注意：**
>您不能使用此API来添加或删除根节点中的条目。 您也无法重命名，移动或删除特殊的`Bookmarks Bar` 和 `Other Bookmarks`。

##### 3.3.3.1 创建书签

要创建书签，请使用`chrome.bookmarks.create`方法。 该方法有两个参数 - `bookmark` (object类型) 和 `callback` (function类型)。 使用书签对象中定义的属性 - `parentId`，`title`和`url` - 您可以创建书签或文件夹。 请注意，如果url为空或缺失，则创建的书签将是一个文件夹。 回调函数接收一个类型为`BookmarkTreeNode`的参数。

![](/assets/3-34.png)

清单3-24包含相应的代码片段，其中使用`bookmark1`对象，首先创建一个文件夹，然后在该文件夹中创建一个书签。 这是使用`callback`中的`result.id`值完成的。 另请参见图3-27，其中包含创建的文件夹和书签。 一个有趣的事情是，创建的文件夹的ID（以蓝色突出显示）为373，从chrome chrome：// bookmarks/＃373可以看出。 您可以通过使用`chrome.bookmarks.get`方法来检查它，如清单3-24所示。 此方法接受一个字符串`id`或这些`id`组成的数组，以及一个`callback`作为参数。 `callback`接收一个`results`参数，它是一个`BookmarkTreeNode`对象的数组。

**Listing 3-24.** Chapter 3 /BookmarksAPI/event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var bookmark1 = {
		title : "MyBookmark1",
		//If url is null or missing, created bookmark will be a folder
		url : ""
	};
	var bookmark2 = {
		title : "MyBookmark2",
		url : "http://www.example.org"
	};
	var queryObject = {
		query : "",
		url : "",
		title : "chrome extensions"
	};
	var queryString = "example url";
	//end-region
	//region {calls}
	console.log(greeting);
	/*
		//result is of type BookmarkTreeNode
		chrome.bookmarks.create(bookmark1,function(result) {
			console.log("Created bookmark with id: " + result.id);
			bookmark2.parentId = result.id;
			chrome.bookmarks.create(bookmark2);
		});
	*/
	/*
		//string or array of string id
		chrome.bookmarks.get("373",function(results) {
			console.log(results); //array of BookmarkTreeNode
		});
	*/
	/*
		//only title and url are supported
		chrome.bookmarks.update("374",{"title":"Example URL"});
	*/
	//string or object query
	chrome.bookmarks.search(queryString,function(results) {
		console.log(results); //array of BookmarkTreeNode
	});
	//end-region

##### 3.3.3.2 更新书签

书签（或文件夹）也可以更新。 允许更新的具体属性包括`title`和`URL`。 请注意，URL仅适用于不是文件夹的书签。 要更新书签，请调用`chrome.bookmarks.update`方法。 此方法使用以下参数：`id` (string类型 ), `changes` (object类型), 和 `callback` (function 类型).

请注意清单3-24中更新方法中传递的字符串ID。 该id对应于图3-27所示的书签（即MyBookmark2）。 图3-28显示了更新后的书签。 如图所示，书签的标题被更新。 这通过将以下对象传递给更新方法：{“title”：“Example URL”}完成。

![](/assets/3-35.png)

##### 3.3.3.3 更新书签搜索书签

要搜索书签，请使用`chrome.bookmarks.search`方法。 该方法以`query`和`callback`为参数。 可以使用字符串或对象进行查询。 在清单3-24中，使用字符串进行查询。 如果要使用对象，则需要定义以下属性：`query`，`url`和`title`。

##### 3.3.3.4 使用书签层次结构

书签API提供了`chrome.bookmarks.getTree`方法来检索整个书签层次结构。 该方法将一个函数作为唯一的参数。 书签树作为数组传递给此函数。 在下面的代码片段中可以看到与此API对应的用法。

请注意，树是此传递的数组的第一个元素。 由于此树是使用节点表示的，所以`children`属性用于访问嵌套文件夹。 在这种情况下，这些文件夹是 Bookmarks Bar 和 Other Bookmarks。 通过迭代嵌套的`BookmarkTreeNode`项（例如，使用`folder.children.forEach`调用），可以访问层次结构中的所有书签。

	chrome.bookmarks.getTree(function(bookmarkTreeAsArray) {
		var bookmarkTree = bookmarkTreeAsArray[0];
		var folders = [];
		if(bookmarkTree.children) {
			bookmarkTree.children.forEach(function(node) {
				if(node.children.length > 0) folders.push(node);
			});
		}
		if(folders.length > 0) {
			folders.forEach(function(folder) {
				folder.children.forEach(function(bookmarkTreeNode) {
					if(bookmarkTreeNode.url) {
					/*use the node*/
					}
				}
			}
		}
	});

##### 3.3.3.5 相关事件

书签API还提供了许多在书签树中发生某些事件时触发的事件。 以下列表包含事件及其说明。 请注意，正如您以前的示例所看到的，要监听这些事件，您需要提供侦听器函数。 它们通过在事件上调用`addListener`来添加。 要了解有关这些事件的更多信息，请访问https://developer.chrome.com/extensions/bookmarks。

- onCreated - 创建书签或文件夹时触发
- onRemoved - 删除书签或文件夹时触发
- onChanged - 当书签或文件夹更改时触发
- onMoved - 当书签或文件夹移动到其他父文件夹时触发
- onChildrenReordered - 当文件夹中的子项的顺序发生改变(比图UUI排序操作)时触发。
- onImportBegan - 当书签导入会话开始时触发
- onImportEnded - 当书签导入会话结束时触发

#### 3.3.4 Downloads API

Downloads API（即chrome.downloads API）用于以编程方式启动，监视，操纵和搜索下载。 它使用download权限。 代码清单3-25包含了第3章“Exercise Files”文件夹中提供的DownloadsAPI扩展程序的代码。

**Listing 3-25.** Chapter 3 /DownloadsAPI/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var downloadOptions = {
		"url" : "http://www.apress.com/downloadable/download/sample/sample_id/1456/",
		"saveAs" : true
	};
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.downloads.download(downloadOptions,function(downloadId) {
		console.log(downloadId);
		/*
		chrome.downloads.pause(downloadId,function() {
			if(!chrome.runtime.lastError) console.log("pause");
		});
		*/
	});
	chrome.downloads.onCreated.addListener(function(downloadItem) {
		console.log("onCreated:");
		console.log(downloadItem);
	});
	chrome.downloads.onErased.addListener(function(downloadId) {
		console.log("onErased:");
		console.log(downloadId);
	});
	chrome.downloads.onChanged.addListener(function(downloadDelta) {
		console.log("onChanged:");
		console.log(downloadDelta);
	});
	//end-region

##### 3.3.4.1 下载文件

要下载文件，请使用`chrome.downloads.download`方法。 该方法采用两个参数——options（Object类型）和callback（function类型）。 与`DownloadItem`对应的id被传递给回调函数。 options对象描述要下载的内容和方法。 它支持以下属性:

- url (string)
- filename (string)
- saveAs (Boolean)
- method ( "GET" or "POST" string)
- headers (array)
- body (string)

**注意：**
>`DownloadItem`是与下载API相关联的类型。 此类型包含一长串属性（了解详细信息请访问https://developer.chrome.com/extensions/downloads＃type-DownloadItem）。 一些重要的属性包括id，filename，mime，startTime，endTime，bytesReceived和totalBytes。

请注意，如果要下载的URL使用HTTPS协议，则将使用`method`,`headers`和`body`属性。 现在，我们来看一个使用下载方法的例子。 对于这个例子，我们将从http://www.apress.com/9781430250531下载源代码文件。 要下载的文件的确切URL显示在清单3-25（在downloadOptions对象中）。

![](/assets/3-36.png)

正如你所看到的那样，调用download方法，并将downloadOptions对象作为第一个参数传递。 使用url和saveAs属性定义downloadOptions对象。 很明显，url是要下载的URL。 将saveAs属性设置为true可以使用文件选择器来允许用户选择一个文件名。 注意图3-29中的“Downloaded by”部分，参考已启动下载的扩展程序。

##### 3.3.4.2 取消或恢复下载

一旦获得了对应于DownloadItem的id（传递给download方法的回调函数），您可以轻松取消，恢复或甚至暂停下载。 相应的调用非常简单：

- chrome.downloads.cancel(integer downloadId, function callback
- chrome.downloads.resume(integer downloadId, function callback
- chrome.downloads.pause(integer downloadId, function callback)

##### 3.3.4.3 打开下载

如果DownloadItem完成，请使用open方法打开下载的文件。 如果它不完整，则打开方法将通过chrome.runtime.lastError返回错误。这个方法只需要一个单一的参数，这就是downloadId。 请注意，此方法需要额外的权限：downloads.open。 

为了简单地将下载的文件显示在文件管理器的文件夹中，需要调用以下方法：chrome.downloads.show（integer downloadId）。 此外，如果唯一的目的是在文件管理器中显示默认的“Downloads”文件夹，则需要调用chrome.downloads.showDefaultFolder（）方法。

##### 3.3.4.4 删除下载
如果DownloadItem完成并要删除下载的文件，请使用chrome.downloads.removeFile方法。 该方法有两个参数——downloadId和callback。 但是，如果所有这些都只是为了从历史记录中清除下载（不删除下载的文件），请改用chrome.downloads.erase方法。

该方法以query对象和callback回调函数为参数。 query对象可以包含很长的属性列表。 一些重要的包括id，startedBefore，startedAfter，endsBefore，endsAfter，filenameRegex和urlRegex。 要查看完整列表，请访问https://developer.chrome.com/extensions/downloads#method-erase。

##### 3.3.4.5 相关事件
Download API提供了一些有用的事件，例如onCreated，onChanged等，可用于在下载开始时或当相应的DownloadItem的属性更改时提供回调。 以下是有关其描述的事件列表。 请注意，您还可以参考清单3-25，查看这些事件与其侦听器函数的确切用法。

- onCreated —Fired with the DownloadItem object when a download begins
- onErased —Fired with the downloadId when a download is erased from history
- onChanged —Fired when any of a DownloadItem’s properties change (except bytesReceived and estimatedEndTime )

#### 3.3.5 History API
HistoryAPI（即chrome.history API）用于与浏览器的页面访问记录进行交互。 使用此API，您可以在浏览器的历史记录中添加，删除和查询URL。 它使用history权限。 清单3-26包含了第3章“Exercise Files”文件夹中提供的HistoryAPI扩展程序的代码。

**Listing 3-26.** Chapter 3 /HistoryAPI/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var tenMinutesAsMilliseconds = 10 * 60 * 1000;
	//getTime returns the number of milliseconds since the epoch
	var currentTimeAsMilliseconds = new Date().getTime();
	//query to filter history using "text", in the past hour
	var query = {
		"text" : "apress",
		"startTime" : currentTimeAsMilliseconds - 6 * tenMinutesAsMilliseconds,
		"endTime" : currentTimeAsMilliseconds,
		"maxResults" : 10
	};
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.history.search(query,function(results) {
		results.forEach(function(result) {
			//result is of type HistoryItem
			console.log(result);
		});
	});
	chrome.history.getVisits({"url" : "http://www.example.org"},function(results) {
		results.forEach(function(result) {
			//result is of type VisitItem
			console.log(result);
		});
	});
	chrome.history.addUrl({"url" : "http://www.example.org"},function() {
		console.log("addUrl");
	});
	chrome.history.deleteUrl({"url" : "http://www.example.org"},function() {
		console.log("deleteUrl");
	});
	
	/*
	chrome.history.deleteAll(function() {
		console.log("deleteAll");
	});
	*/
	//end-region

要使用query搜索历史记录，请使用`chrome.history.search`方法。 该方法有两个参数 - query（Object类型）和callback（function类型）。qeury对象支持以下属性：

- text —用于查询历史服务的任意文本。 将其留空以检索所有页面。
- startTime —将结果限制在此日期之后访问的人员，以毫秒为单位。
- endTime —在此日期之前访问的人员的结果。 以毫秒为单位。
- maxResults —要检索的最大结果数。 默认为100。

请注意，回调函数接受一个HistoryItem结果数组。 这里，HistoryItem是封装历史查询结果的一个对象。 它支持以下有用的属性id，url，title，lastVisitTime，visitCount等。要检索有关访问URL的信息，请使用chrome.history.getVisits方法。 如清单3-26所示，此方法将一个对象（以url属性）作为其第一个参数。 第二个参数是一个接收VisitItem结果数组的回调函数。

**注意：**
>一个VisitItem是一个封装了一个URL访问的对象。 它由以下属性组成：id，visitId，visitTime，referVisitId和有着TransitionType（描述浏览器导航到特定URL的方式）的transition。 您可以访问https://developer.chrome.com/extensions/history#transition_types了解有关TransitionType的更多信息。

![](/assets/3-37.png)

##### 3.3.5.1 添加和删除URL

要在当前时刻向历史记录添加URL，请使用`chrome.history.addUrl`方法。 该方法需要两个参数 - details（Object类型）和callback（function类型）。 details对象只支持url属性。 同样，可以使用`chrome.history.deleteUrl（Object，callback）`方法，从历史记录中删除给定URL的所有历史记录。 请注意，要从历史记录中删除所有项目，请使用`chrome.history.deleteAll`方法。 这个方法只需要一个回调函数作为它的参数。

##### 3.3.5.2 相关事件

history API支持onVisited和onVisitRemoved事件。 访问URL时触发onVisited事件，在相应的侦听器函数中提供该URL的HistoryItem数据。 请注意，此事件在页面加载之前触发。 同样，当从历史记录服务中删除一个或多个URL时，会触发onVisitRemoved事件。 与此事件对应的侦听器函数接收一个对象。 此对象支持以下属性：

- allHistory (Boolean)-如果所有历史记录都被删除则该属性置为true，此时urls将会是空。
- urls (array)-删除的历史记录组成的字符串数组。

#### 3.3.6 Notifications API
您可以使用chrome.notifications API并使用模板创建丰富的通知，并向系统托盘中的用户显示这些通知。 它使用`notifications`权限。 清单3-27包含了第3章“Exercise Files”文件夹中提供的NotificationsAPI扩展程序的代码。

**Listing 3-27.** Chapter 3 /NotificationsAPI/ event_script.js
	
	//region {variables and functions}
	var greeting = "Hello World!";
	var title = "NotificationsAPI";
	var message = "Test message X";
	var oneMinuteAsMilliseconds = 1 * 60 * 1000;
	//getTime returns the number of milliseconds since the epoch
	var currentTimeAsMilliseconds = new Date().getTime();
	var notificationId = "myNotification1";
	var NOTIFICATION_TEMPLATE_TYPE = {
		BASIC : "basic",
		IMAGE : "image",
		LIST : "list",
		PROGRESS : "progress"
	};
	var myButton1 = {
		title : "Click X",
		iconUrl : "button.png"
	};
	var myButton2 = {
		title : "Click Y",
		iconUrl : "button.png"
	};
	var myItem1 = {
		title : "Item X",
		message : "This is item X"
	};
	var myItem2 = {
		title : "Item Y",
		message : "This is item Y"
	};
	var notificationOptions = {
		type : NOTIFICATION_TEMPLATE_TYPE.LIST,
		iconUrl : "icon.png",
		title : title,
		message : message,
		eventTime : currentTimeAsMilliseconds + oneMinuteAsMilliseconds,
		buttons : [myButton1,myButton2],
		/*imageUrl : "icon.png",*/
		items : [myItem1,myItem2], //comment out for BASIC
		/*progress : 0,*/
		isClickable : true
	};
	//end-region
	//region {calls}
	console.log(greeting);
	chrome.notifications.create(notificationId,notificationOptions,
		function(id) {
			console.log("create: " + id);
		}
	);
	/*
	chrome.notifications.clear(notificationId,function(wasCleared) {
		console.log("clear: " + wasCleared);
	});
	*/
	/*
	chrome.notifications.getAll(function(notifications) {
		console.log("getAll:");
		console.log(notifications);
	});
	*/
	chrome.notifications.onClicked.addListener(function(id) { //notification-id
		console.log("onClicked: " + id);
		notificationOptions.title = title + " (onClicked)";
		chrome.notifications.update(notificationId,notificationOptions,
			function(wasUpdated) {
				console.log("update: " + wasUpdated);
			}
		);
	});
	chrome.notifications.onClosed.addListener(function(notificationId,byUser) {
		console.log("onClosed: " + notificationId);
	});
	chrome.notifications.onButtonClicked.addListener(
		function(notificationId,buttonIndex) {
			console.log("onButtonClicked: " + buttonIndex);
		}
	);
	//end-region

##### 3.3.6.1 创建和清除通知

要创建通知，请使用`chrome.notifications.create`方法。此方法使用三个参数 - notificationId（String类型）, notificationOptions（Object类型）和callback。请注意，notificationOptions对象描述通知的内容。其支持的完整属性列表可以在https://developer.chrome.com/apps/notifications#type-NotificationOptions中找到。

notificationOptions对象中的一个重要属性是type，该属性声明了用于创建通知的模板。其它诸如title , message , buttons , imageUrl , items , progress等属性定义了模板的具体部分。图3-30至3-32包含不同通知模板的示例。 在清单3-27中，注意用作选择通知模板类型的枚举`NOTIFICATION_TEMPLATE_TYPE`对象。

您可以通过调用`chrome.notifications.clear`方法来清除通知，该方法将notificationId作为其参数。 要清除所有通知，请使用clear方法，以及chrome.notifications.getAll方法，它将回调函数作为参数。 这个回调函数接收一个包含所有通知的对象。

##### 3.3.6.2 更新通知

要更新现有通知，请使用chrome.notifications.update方法。 此方法使用三个参数 - notificationId（类型String）,notificationOptions（Object类型）和回调函数。 回调函数接收一个布尔参数，指示是否更新通知。 在清单3-27中，请注意，在onClicked事件的侦听器函数中调用update方法。

##### 3.3.6.3 相关事件

通知API支持onClosed，onClicked和onButtonClicked事件。 通知关闭时，系统或用户操作会触发onClosed事件。 如清单3-27所示，与此事件相对应的侦听器函数接收两个参数 - notificationId（string）和byUser（Boolean）。

当用户单击通知中的按钮（参见图3-34）时，onButtonClicked事件触发。 请注意，按钮是使用notificationOptions对象中的buttons属性提供的（参见清单3-27）。 该事件的侦听器函数接收notificationId以及buttonIndex。 同样，当用户点击通知的非按钮区域时，onClicked事件触发。 清单3-27中使用了此事件来更新通知的标题属性（参见图3-33）。

#### 3.3.7 Storage API

storageAPI（即，chrome.storage API）用于存储，检索和跟踪对用户数据的更改。 它使用storage权限。 清单3-28包含第3章“Exercise Files”文件夹中提供的StorageAPI扩展程序的代码。 该API经过优化，可满足扩展的特定存储需求。 它提供与localStorage API相同的存储功能，具有以下主要区别：

- 用户数据可以自动与Chrome同步（使用chrome.storage.sync API）同步。
- 您的扩展程序的内容脚本可以直接访问用户数据，而无需背景页面。
- 与阻塞和串行localStorage API相比，批量读写操作是异步的，因此更快。
- 用户数据可以存储为对象（localStorage API将数据存储在字符串中）。

##### 3.3.7.1 同步与本地存储
要存储扩展程序的用户数据，可以使用storage.sync或storage.local API。 使用storage.sync时，只要用户已启用同步，存储的数据将自动同步到用户登录的任何Chrome浏览器。

当Chrome离线时，Chrome会将数据存储在本地。 下次浏览器在线时，Chrome会同步数据。 即使用户禁用同步，storage.sync API仍然可以工作。 在这种情况下，它将与storage.local API的行为相同。

**注意：**
>机密用户信息不应存储！ 存储区域是未加密的。

**Listing 3-28.** Chapter 3 /StorageAPI/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	//end-region
	//region {calls}
	console.log(greeting);
	/*
	//single key or a list of keys for items to remove
	chrome.storage.sync.remove("color",function() {
		console.log("remove");
		chrome.storage.sync.get("color",function(items) {
			console.log("get");
			console.log(items);
		});
	});
	*/
	chrome.storage.sync.set({"color":"red"},function() {
		console.log("set");
		//string or array of string or object keys
		chrome.storage.sync.get("color",function(items) {
			console.log("get");
			console.log(items);
		});
	});
	chrome.storage.onChanged.addListener(function(changes,areaName) {
		console.log(changes);
		//"sync","local" or "managed"
		console.log(areaName);
	});
	//end-region

##### 3.3.7.2 设置和获取项目

您可以使用以下API调用chrome来设置和从存储中获取项目。 `storage.sync.set`和`chrome.storage.sync.get`。 get方法将字符串，字符串数组或对象键作为其第一个参数，set方法将一个对象（键值对）作为其第一个参数。 get方法中的第二个参数是一个回调函数，它接收一个对象作为参数。图3-35包含来自这些方法调用的相应日志。

##### 3.3.7.3 删除项目

您可以通过调用chrome.storage.sync.remove方法轻松地删除存储中的项目。此方法将字符串或字符串组成的数组作为其第一个参数，并且可选的回调函数作为其第二个参数。要从存储中删除所有项目，请使用`chrome.storage.sync.clear`方法。

##### 3.3.7.4 相关事件

storage API提供了chrome.storage.onChanged事件。当一个或多个（存储）项目更改时，此事件将触发。与此事件对应的侦听器函数接收两个参数 - changes（Object）和 areaName（string）。这里，areaName是存储区域的名称（sync，local等）。

#### 3.3.8 选项卡API
选项卡API（即chrome.tabs API）用于与浏览器的选项卡系统进行交互。 您可以使用此API在浏览器中创建，修改和重新排列选项卡。 它使用选项卡权限。 清单3-29包含了第3章“Exercise Files”文件夹中提供的Tabs API扩展程序的代码。 您已经使用了query，connect，sendMessage，executeScript和insertCSS方法。 本节将介绍其余的方法。

您可以使用大多数chrome.tabs方法和事件，而无需在扩展程序的manifest.json文件中声明任何权限。 但是，如果您需要访问tabs.Tab的url，title或favIconUrl属性，则必须在manifest.json文件中声明tabs权限。

Listing 3-29. Chapter 3 /TabsAPI/ event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var createProperties = {
		url : "http://www.example.org",
		active : false,
	};
	var updateProperties = {
		pinned : true
	};
	function getJavaScriptCode(dataUrl) {
		var javascriptCode = "var imgElement = document.createElement('img');";
		javascriptCode += "document.body.appendChild(imgElement);";
		javascriptCode += "imgElement.style.borderTop = '2px dashed silver';";
		javascriptCode += "imgElement.src = ";
		javascriptCode += "'" + dataUrl + "';";
		return javascriptCode;
	}
	function createAndUpdateTab(tab) {
		chrome.tabs.create(createProperties,function(tab) {
			console.log("create");
			//integer or array of integers
			//chrome.tabs.remove(tab.id);
			/*
			chrome.tabs.duplicate(tab.id,function(tab) {
				console.log("duplicate");
			});
			*/
			chrome.tabs.update(tab.id,updateProperties,function(tab) {
				console.log("update");
				//chrome.tabs.reload(tab.id);
				chrome.tabs.getZoom(tab.id,function(zoomFactor) {
					console.log("getZoom");
					console.log(zoomFactor); //1
				});
				/*
				chrome.tabs.setZoom(tab.id,2,function() {
					console.log("setZoom");
				});
			*/
			});
		});
	}
	//end-region
	
	//region {calls}
	console.log(greeting);
	chrome.browserAction.onClicked.addListener(function(tab) {
		chrome.tabs.captureVisibleTab({"format":"png"},function(dataUrl) {
			//Cannot access a chrome:// URL
			chrome.tabs.executeScript(tab.id,{"code":getJavaScriptCode(dataUrl)});
		});
		//createAndUpdateTab(tab);
	});
	//end-region

##### 3.3.8.1 创建和删除选项卡

要创建选项卡，请使用`chrome.tabs.create`方法。 这个方法有两个参数：createProperties（object类型）和callback（function类型）。 createProperties对象支持以下有用的属性 - index，url，active和pinned。 回调函数接收一个Tab（Tabs.Tab）类型的参数。 还可以通过调用chrome.tabs.duplicate方法来复制选项卡，如清单3-29所示（请参阅createAndUpdateTab函数中的注释部分）。 使用chrome.tabs.remove方法关闭一个或多个选项卡。 此方法使用该选项卡的ID或此类ID的列表作为其第一个参数来关闭它。 它还支持可选的回调函数作为其第二个参数。
##### 3.3.8.2 更新选项卡

使用chrome.tabs.update方法修改选项卡的属性。 该方法有三个参数 - tabId，updateProperties和callback。 您可能已经猜到，tabId是要更新的标签的ID。 回调函数通过tab参数接收更新的选项卡的详细信息。 updateProperties对象指定要更新的属性。 只支持以下属性：url，active，highlight，pinned，quieted和openerTabId。 请注意，您还可以通过调用chrome.tabs.reload方法重新加载选项卡。 此方法将标签的ID作为其第一个参数重新加载，可选的回调函数作为其第二个参数。

**Listing 3-30.** Chapter 3 /TabsAPI/ manifest.json

	{
		"manifest_version" : 2,
		"name" : "API Demos: tabs API",
		"description" : "Demonstrates use of the tabs API",
		"version" : "1.2",
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		},
		"browser_action" : {
			"default_icon" : "icon.png"
		},
		"permissions" : [
			"tabs",
			"<all_urls>"
		]
	}

**注意：**

>要缩放指定的选项卡，请使用chrome.tabs.setZoom方法。 另外，要获取指定选项卡的当前缩放系数，请使用chrome.tabs.getZoom方法。

##### 3.3.8.3 捕获选项卡

要捕获指定窗口中当前活动选项卡的可见区域，请使用chrome.tabs.captureVisibleTab方法。 这个方法需要三个参数windowId，options和callback。 windowId参数是可选的，它默认为当前窗口。 options参数用于指定图像的格式。 回调函数接收dataUrl（string）参数，该参数是引用包含捕获选项卡的可见区域的编码图像的数据的URL。 请注意，dataUrl可以分配给HTML img元素的src属性，如清单3-29所示。 与此对应的图像如图3-36所示。 另请注意，此方法需要一个额外的权限 - <all_urls> - 如清单3-30所示。

##### 3.3.8.4 相关事件

chrome.tabs API提供了与几乎所有可用方法相对应的列表。 例如，onCreated，onUpdated，onRemoved，onZoomChange等事件。 有关此类事件的完整列表，请访问https://developer.chrome.com/extensions/tabs。

#### 3.3.9 XHR API
扩展程序可以通过请求跨域权限与远程服务器通信（使用XMLHttpRequest对象）。 这样的权限需要匹配模式的帮助，匹配模式提供对一个或多个主机的访问（参见清单3-31中的“*：// localhost / *”）。

**Listing 3-31.** Chapter 3 /XHRAPI/manifest.json

	{
		"manifest_version" : 2,
		"name" : "API Demos: xhr API",
		"description" : "Demonstrates use of the xhr API",
		"version" : "1.2",
		"background" : {
			"scripts" : ["event_script.js"],
			"persistent" : false
		},
		"permissions" : [
			"*://localhost/*"
		]
	}


一旦请求了主机许可，就可以与常规网页中使用的方式类似使用XHR API了。 在清单3-32中，请注意，我的本地HTTP服务器提供了ucService（将输入字符串转为大写的PHP脚本）。 PHP脚本可以在清单3-33中找到。

**Listing 3-32.** Chapter 3 /XHRAPI/event_script.js

	//region {variables and functions}
	var greeting = "Hello World!";
	var xhr = new XMLHttpRequest();
	function onReadyStateChange() {
		if(xhr.readyState == 4) {
			console.log(xhr.responseText);
		}
	}
	var host = "http://localhost/";
	var ucService = host + "Exercise Files/Chapter3/XHRAPI/webpage.php?";
	var queryString = "name=" + encodeURIComponent("xhr api");
	//end-region
	//region {calls}
	console.log(greeting);
	xhr.onreadystatechange = onReadyStateChange;
	xhr.open("GET",ucService + queryString);
	xhr.send();
	//end- region

查询字符串“name = xhr api”以静态方式提供。 它可以在您的扩展中动态提供，例如，从浏览器操作弹出窗口或Content-UI等。图3-37显示了来自服务器的相应响应。

**Listing 3-33.** Chapter 3 /XHRAPI/WebServer/webpage.php

	<?php
		$raw_input = trim($_GET["name"]);
		if(!preg_match("/^[a-zA-Z ]*$/",$raw_input)) {
			echo "";
		} else {
			echo strtoupper($raw_input);
		}
	?>

### 3.4 总结

本章的开头讨论了前一章节的输入组件 - 多功能输入框，上下文菜单项和Content-UI。之后，您了解了脚本组件可以使用Google Chrome Extensions框架提供的消息传递API彼此交互的各种方式。除此之外，您还了解了普通网页如何与扩展程序进行交互。然后，在了解如何使用扩展框架提供的许多有用的API之前，了解清单文件中的哪些权限字符串。您还学习了如何在扩展名中使用XHR，可以​​使用清单文件中的匹配模式权限字符串来跨域。

在下一章中，您将阅读Chrome扩展框架提供的一些其他功能，例如主题，覆盖页面（覆盖新选项卡，书签页面等）以及选项页面，从而可以在Chrome浏览器上增强您的扩展程序。





