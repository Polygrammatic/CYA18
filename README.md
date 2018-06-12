

<h1 align="center">Christo</h1>

<div align="center">
	<strong>Yousician Programming Assignment</strong>
</div>
<div align="center">
	Using <code>Unity 2018.1.0f2</code>

</div>
<br />

<div align="center">
	<h3>
		<a href="http://developer.yle.fi/api_docs.html">
			Yle API Docs
		</a>
		<span> | </span>
		<a href="http://nsubstitute.github.io/">
			NSubstitute
		</a>
		<span> | </span>
		<a href="https://github.com/modesttree/Zenject">
			Zenject
		</a>
		<span> | </span>
		<a href="https://docs.unity3d.com/Manual/UnityWebRequest.html">
			UnityWebRequest
		</a>
		<span> | </span>
		<a href="https://github.com/neuecc/UniRx">
			UniRX
		</a>
		<span> | </span>
		<a href="https://simplejson.readthedocs.io/en/latest/">
			SimpleJSON
		</a>
	</h3>
</div>

<div align="center">
	<sub>
		More information about Christo ❤︎ and the projects worked on:
		<a href="http://www.polygrammatic.com/">Media appearances</a>
		<br>
			<a href="https://yousician.com/">Yousician.com</a>
		</div>

## Table of contents

- [The Task](#the-task)
- [Process Flow](#process-flow)
- [Unity Editor](unity-editor)
	- [Unit Tests](#unit-tests)
- [URL](#URL)
	- [Button Creating A Search](#button-creating-a-search)
	- [Request Builder](#request-builder)
	- [Request Handler](#request-handler)
	- [URL Manager](#url-manager)
	- [Deserialize Response](#deserialize-response)
- [User Interface](#user-interface)
	- [ScrollBar Creating A Search](#scrollbar-creating-a-search)
	- [Loading Wheel](#loading-wheel)
	- [User Interface Manager](#user-interface-manager)
- [Technology](#technology)
	- [Zenject Installers](#zenject-installers)
	- [UniRx](#unirx)
	- [Namespace](#namespace)
- [Links](#links)

# The Task

### The task was to search programs from the Yle API:
<img src="https://i.imgur.com/VJeCB1S.gif">

- The application should have an input field where the user can provide what programs to search for (a search query).
- The **Finnish title** of each result should be displayed in a scrollable list.
- When submitting a search, only the first 10 results should be retrieved from the API.
- When the user scrolls to the last few items in the list, the next 10 results should be appended to the list. An example for this type of scrolling can be seen on the desktop website of Facebook.
- Pressing on the row should display more details about the programme. For this you can select any 5 relevant fields from the JSON.
- Technologies to use: Unity UI, UnityWebRequest, Any JSON library you prefer.

# Process Flow
<img src="https://i.imgur.com/FtXJNuw.png">

# Unity Editor
<img src="https://i.imgur.com/M0wNTi4.png">
<br>
On public variables that are set in `Inspector` visual notation on fields not coloured by Visual Studio are removed.

Code is attached to an empty `GameObject` titled `Main`.

## Unit Tests

Unit Tests are provided with code using [NSubstitute](http://nsubstitute.github.io/help/getting-started/) and [NUnit](https://github.com/nunit/docs/wiki/Attributes).


# URL

## Button Creating A Search
When `public Button searchButton;` is pressed, `NewSearchMade.cs` uses `Button` to call 3x methods, and passes `private int _offset = 0;` into the `UrlRequestBuilder.cs`, which builds the sent request:
```csharp
private void Start()
{
	searchButton.onClick.AddListener(delegate
	{
		_urlRequestBuilder.CheckValidBuildRequest(_offset);
		_urlManager.ResetElementCreatedCounter();
		_userInterfaceManager.ClearElements();
	});
}
```
The offset is set to `0` to make sure that all requests made by the search button have an offset of `0` as scrolling to the bottom of the window will use a different offset `offset += 10`.
*Example:* `url/endpoint?app_id&app_key&offset=0`.


## Request Builder
`UrlRequestBuilder.cs` recieves the request from both the search button and the scroll bar reaching the bottom of the `Scroll Rect` container.

### `CheckValidBuildRequest()`
Will clear any previous notifications, then check if a valid search query is entered. If not, a notification is presented to the user via `searchQueryNotFound.SetActive(true);` which is a `GameObject` with a `Text` component attached. If search query isn't empty, spaces are removed, then `BuildRequest(lOffset);` is called.

The `_offset` from `NewSearchMade.cs` is passed aswell. When a request is made from the `ScrollBar`, a different `_offset` will be brought in.

### `BuildRequest(int lOffset)`

```csharp
public void BuildRequest(int lOffset)
{
	string lsQuery = searchQuery.text;
	string lsUrlRaw = string.Format("https://{0}/{1}?app_id={2}&app_key={3}&q={4}&limit={5}&offset={6}",
	_urlComponents[0], _urlComponents[1], _urlComponents[2], _urlComponents[3], lsQuery, miLimit, lOffset);
	string lsUrlEncoded = lsUrlRaw.Replace(" ", "%20"); // remove spaces

	SendRequestToHandler(lsUrlEncoded, miElementsToSpawn);
}
```
The entered search query from the user is stored in `lsQuery` where it is added to `string lsUrlRaw`, *non-encoded string* holding `_urlComponents[0]`, `_urlComponents[1]`, `_urlComponents[2]`, `_urlComponents[3]`, `lsQuery`, `miLimit`, and `lOffset`.

`_urlComponents` is a `private string[]` holding `URL`, `Endpoint`, `App ID`, and `App Key`.

`lsUrlRaw` may contain spaces, so needs to be encoded to be URL friendly. Spaces are removed using `Replace()` and are replaced with `%20`. Encoded string is sent to `SendRequestToHandler(lsUrlEncoded, miElementsToSpawn)`.

`int miElementsToSpawn = 10;` as we only require 10 elements to be spawned at any given time, providing there are `10` left to spawn.

### `SendRequestToHandler(string lsUrlEncoded, int liElementsToSpawn)`
```csharp
private void SendRequestToHandler(string lsUrlEncoded, int liElementsToSpawn)
{
	StartCoroutine(_urlRequestHandler.SendRequest(lsUrlEncoded, miElementsToSpawn));
	_urlManager.rbRequestCompleted.Value = false;
}
```
A `Coroutine` starts taking arguments given, and UniRx `BoolReactiveProperty` is set to false to that we know a request is taking place. This `ReactiveProperty` is used for the `LoadingWheel` which is `SetActive` `true` and `false` in `ScrollBarReachesBottom.cs` by using UniRx `.Subscribe()`.

*Note: I haven't used UniRx before, but after learning & researching, I would replace the coroutine with UniRx ObervableWWW.Get(), but used UnityWebRequest instead as is required by the task.*


## Request Handler
`UrlRequestHandler.cs` is where the `IEnumerator` is located for sending the web request using `UnityWebRequset`. When a request is complete, it is sent to `UrlManager.cs` to be `ToDeserializsed` using `SimpleJSON` Json library.

```csharp
public IEnumerator SendRequest(string lsReq, int lElementsToSpawn)
{
	// created interface for unitywebrequest for unittesting
	using (var www = _webRequestFactory.Create(lsReq))
	{
		yield return www.SendWebRequest();

		string lsResult =  www.isNetworkError || www.isHttpError ? www.error : www.downloadHandler.text;
		_urlManager.RequestCompleted(lsResult, lElementsToSpawn);
	}
}
```

**For Unit Testing**: `IWebRequest.cs` has 2x interfaces, `IWebRequest` and, `IWebRequestFactory`,
```csharp
public interface IWebRequest : IDisposable
{
	bool isNetworkError { get; }
	bool isHttpError { get; }
	string error { get; }
	DownloadHandler downloadHandler { get; set; }

	UnityWebRequestAsyncOperation SendWebRequest();
}
```
```csharp
public interface IWebRequestFactory
{
	IWebRequest Create(string lString);
}
```
and 2x classes.
```csharp
public class UnityWebRequestFactory : IWebRequestFactory
{
	public IWebRequest Create(string lString)
	{
		var unityWebRequest = UnityWebRequest.Get(lString);
		return new UnityWebRequestWrapper(unityWebRequest);
	}
}
```
```csharp
public class UnityWebRequestWrapper : IWebRequest
{
	readonly private UnityWebRequest _unityWebRequest;

	public UnityWebRequestWrapper(UnityWebRequest lUnityWebRequest)
	{
		_unityWebRequest = lUnityWebRequest;
	}

	public DownloadHandler downloadHandler
	{
		get
		{
			return _unityWebRequest.downloadHandler;
		}
		set
		{
			_unityWebRequest.downloadHandler = value;
		}

	}

	public string error
	{
		get
		{
			return _unityWebRequest.error;
		}
	}

	public bool isHttpError
	{
		get
		{
			return _unityWebRequest.isHttpError;
		}
	}

	public bool isNetworkError
	{
		get
		{
			return _unityWebRequest.isNetworkError;
		}
	}

	public void Dispose()
	{
		_unityWebRequest.Dispose();
	}

	public UnityWebRequestAsyncOperation SendWebRequest()
	{
		return _unityWebRequest.SendWebRequest();
	}
}
```
Allowing for testing in `UrlRequestHandlerTests.cs`.

`GivenValidRequestString_WhenWebRequestIsNetworkError_ThenErrorReturned()`
`GivenValidRequestString_WhenWebRequestIsHttpError_ThenErrorReturned()`
`GivenValidRequestString_WhenNoError_ThenApiReturned()`

## URL Manager

### `UrlManager.cs`
The URL Manager:
- Ensures all required components are present using `[RequireComponent(typeof())]` in Unity
- Holds UniRx `ReactiveProperty` which declares to other classes if a request is complete or not
- Counts remaining element(s) needed to be created
```csharp
public int CountElementsLeftToSpawn()
{
	return _miElementsLeftToSpawnCount;
}
```
- Resets the element(s) created counter
```csharp
public void ResetElementCreatedCounter()
{
	_miCountElementsCreated = 0;
}
```
- Processes completed request using `UrlDeserialize.cs` which deserializes the response using `SimpleJSON` and returns a list of `ProgramInformation` from concrete class `ProgramInformation.cs` in a list called `programInformationCollection`
- Passes information to `UserInterfaceManager.cs` about which elements to spawn, and information relating to the elements.

```csharp
public void RequestCompleted(string lsToDeserializse, int liElementsToSpawn)
{
	var collectionObject = _urlDeserialize.DeserializeResponse(lsToDeserializse);
	int liCountElementsLeftToSpawn = int.Parse(collectionObject.ElementAt(0).MetaCount);

	rbRequestCompleted.Value = true;

	// Parent + Child = 2, only want to count 1, so CountInstantiatedElements() / 2
	_miElementsLeftToSpawnCount = liCountElementsLeftToSpawn - (_uiManager.CountInstantiatedElements() / 2);

	// create elements
	if (_miElementsLeftToSpawnCount != 0)
	{
		_uiManager.NoSearchResults(false);
		for (int i = 0; i < _miElementsLeftToSpawnCount && i < 10; i++)
		{
			_uiManager.CreateElements(_miCountElementsCreated, collectionObject.ElementAt(i));
			_miCountElementsCreated++;	
		}
	}
	else if (liCountElementsLeftToSpawn == 0) _uiManager.NoSearchResults(true);
}
```
`i < _miElementsLeftToSpawnCount && i < 10;` ensures that the request is either under `10` or under the remaining elements left to spawn count. If there are `5` elements left, `5` will be spawned, not `10`.

`_uiManager.CountInstantiatedElements()` returns all elements (parent and children), so needs to be `/2` to get the correct amount of element pairs created.



## Deserialize Response
`UrlDeserializeResponse.cs` uses `SimpleJSON` to process the returned string from `UnityWebRequest`.

The task was to return the Finnish title, and "select any 5 relevant fields from the JSON". The Swedish title is also returned incase the Finnish title `== string.Empty`, which in some instances is true.
```csharp
public IEnumerable<ProgramInformation>DeserializeResponse(string lsResults)
{
	var parseRawJson = JSON.Parse(lsResults);
	var programInformationCollection = new List<ProgramInformation>();

	for (int i = _urlRequest.miOffset; i < _urlRequest.miOffset + _urlRequest.miLimit; i++)
	{
		programInformationCollection.Add(new ProgramInformation
		{
			MetaCount = parseRawJson["meta"]["count"],
			FiTitle = parseRawJson["data"][i]["title"]["fi"],
			SvTitle = parseRawJson["data"][i]["title"]["sv"],
			Id = parseRawJson["data"][i]["id"],
			Type = parseRawJson["data"][i]["type"],
			VideoType = parseRawJson["data"][i]["video"]["type"],
			TypeMedia = parseRawJson["data"][i]["typeMedia"],
			Collection = parseRawJson["data"][i]["collection"]
		});
	}

	return programInformationCollection;
}
```
Concrete class `ProgramInformation.cs` was created:
```csharp
public class ProgramInformation
{
	public string MetaCount { get; set; }
	public string FiTitle { get; set; }
	public string SvTitle { get; set; }

	public string Id { get; set; }
	public string Type { get; set; }
	public string VideoType { get; set; }
	public string TypeMedia { get; set; }
	public string Collection { get; set; }
}
```

# User Interface
## ScrollBar Creating A Search
`ScrollBarReachesBottom.cs` is similar to `NewSearchMade.cs` but has an offset of `10`, and requires conditions to be met before sending a request using UniRx `BoolReactiveProperty`.
```csharp
private void ScrollBarConditionsMet()
{
	if (_scrollBar.activeSelf == true && _sb.value != 1.0f && _sb.value < _sb.size / 1.5) _rbScrollBarConditionsMet.Value = true;
	else _rbScrollBarConditionsMet.Value = false;
}
```
The scroll bar needs to be active, and the position needs to `!= 1.0f` as the first return might trigger a further search. The scroll bar also needs to be less than its size divided by `1.5` (two thirds). This ensures that as the list of elements gets longer, the next `10` are triggered to be created in a position relative to the bottom without having to use a viewport system to check how close it is to the bottom.

Using UniRx `.Subscribe()` the `LoadingWheel` is shown and hidden depending on the conditions of the request declaratively rather than using `Unity Update()`, and calls `NextUrlRequest()` method.
```csharp
_urlManager.rbRequestCompleted.Subscribe(
		b => { if (b) _loadingWheel.SetActive(false); else if (!b) _loadingWheel.SetActive(true); });

_rbScrollBarConditionsMet.Subscribe(
		b => { if (b && _urlManager.rbRequestCompleted.Value == true) NextUrlRequest(); });
```
`NextUrlRequest()` sends a build request with a starting offset of `10`, then updates the offset using `+= 10` after every call. Scroll bar conditions are also set to false to make sure that another call is not made.
```csharp
private void NextUrlRequest()
{
	if (_urlManager.CountElementsLeftToSpawn() != 0)
	{
		_urlRequestBuilder.BuildRequest(_miUrlOffest);
		_miUrlOffest += 10;
		_rbScrollBarConditionsMet.Value = false;
	}
}
```


## Loading Wheel
The loading wheel has a `sprite[]`  with cycles single frames created in PhotoShop into `Image` component on an Update loop which only runs when the object is active.
```csharp
gameObject.GetComponent<Image>().sprite = _sprites[_miCurrentImageIndex];

if (_timeRemaining > 0.0f)
{
	_timeRemaining -= Time.deltaTime;
	if (_miCurrentImageIndex == 8) _miCurrentImageIndex = 0;
}
else if (_timeRemaining < 0.0f)
{
	_miCurrentImageIndex += 1;
	_timeRemaining = _framesPerSecond;
}
```

## User Interface Manager
The `UserInterfaceManager.cs` is called from Url Manager and takes the count of spawned elements, and the ProgramInformation element that is being created.

In the Unity Inspector, a reference to the parent and child `prefabs` are set, as well as the reactive container location in Hierarchy and a `GameObject` that holds the `Text` for no results found. 2x GameObjects will be created for each request, a parent, and a child, which will remain hidden until the parent is selected. 

For this program, you will be able to open multiple at once similar to how `RES` works on `Reddit` for images.
`InstantiateElementPair()` is called first from `CreateElements()` using `Instantiate`, and renames the parent and child objects appropriately and sequentially. The element pair is places in the reactive container.
```csharp
private void InstantiateElementPair(int lLoopPosition, ProgramInformation lElement)
{
	GameObject lParent = Instantiate(mgoParent);
	lParent.name = string.Format("Parent_{0}:{1}", lLoopPosition, lElement.Id);	// rename parent element
	lParent.transform.SetParent(mgoSpawnLocation.transform, false);				// put in container

	GameObject lChild = Instantiate(mgoChild);
	lChild.name = string.Format("Child_{0}:{1}", lLoopPosition, lElement.Id);	// rename child element
	lChild.transform.SetParent(mgoSpawnLocation.transform, false);              // put in container
	lChild.SetActive(false);													// start children hidden
}
```
After the element pair has been `Instantiated`, they are found where they can be processed. Their colour is set, text updated and titles amended appropriately. If there is no Finnish title available, the Swedish title is applied alongside a notification.

```csharp
private void ProcessParentElement(Text lParentText, ProgramInformation lElement)
{
	lParentText.color = new Color(0.1f, 0.1f, 0.1f, 1.0f);
	lParentText.transform.name = lElement.FiTitle;
	lParentText.text = lElement.FiTitle;

	string lsNoFiTitle = "Result has no Finnish title. Haulla ei löytynyt suomen kielistä otsikko.";

	if (lParentText.text == "" || lParentText.text == " ")
	{
		if (lElement.SvTitle != "" || lElement.SvTitle != " ") lParentText.text = string.Format("{0} \nSwedish: {1}", lsNoFiTitle, lElement.SvTitle);
		else if (lElement.SvTitle == "" || lElement.SvTitle == " ") lParentText.text = lsNoFiTitle;

		lParentText.fontStyle = FontStyle.Italic;
		lParentText.color = new Color(0.2f, 0.2f, 0.2f, 1.0f);
	}
}
```
The child element is processed in a similar manner to the parent whereby the text is updated.
```csharp
private void ProcessChildElement(Text lChildText, ProgramInformation lElement)
{
	lChildText.text = string.Format("Id: {0}\nType: {1}\nTypeMedia: {2}\nVideoType: {3}\nCollection: {4}",
		lElement.Id, lElement.Type, lElement.TypeMedia, lElement.VideoType, lElement.Collection);
}
```
The `CreateElements()` method calling `InstantiateElementPair()`, `ProcessParentElement()` and `ProcessChildElement()`.

Once the elements have been processed, button functionality is applied with `Dictionary< string, bool >();` using `Delegates`.
```csharp
public void CreateElements(int liSpawnedElementCount, ProgramInformation lElement)
{
	// create parent and child element
	InstantiateElementPair(liSpawnedElementCount, lElement);

	// find created elements
	GameObject lgoFoundNewParent = GameObject.Find(string.Format("/Canvas/Results/ContainerFixed/ContainerReactive/Parent_{0}:{1}", liSpawnedElementCount, lElement.Id ));
	GameObject lgoFoundNewChild = GameObject.Find(string.Format("/Canvas/Results/ContainerFixed/ContainerReactive/Child_{0}:{1}", liSpawnedElementCount, lElement.Id));

	Text lParentText = lgoFoundNewParent.gameObject.GetComponentInChildren<Text>();
	Text lChildText = lgoFoundNewChild.gameObject.GetComponentInChildren<Text>();

	// change the text, name and details of parent
	ProcessParentElement(lParentText, lElement);
	ProcessChildElement(lChildText, lElement);

	// Get parent button
	Button _foundNewParentButton = lgoFoundNewParent.GetComponent<Button>();

	var dictionary = new Dictionary< string, bool >();

	dictionary[lgoFoundNewParent.ToString()] = true;
		
	_foundNewParentButton.onClick.AddListener(delegate 
	{
		if (dictionary[lgoFoundNewParent.ToString()])
		{
			TitleElementClicked(lgoFoundNewParent, lgoFoundNewChild, true);
			dictionary[lgoFoundNewParent.ToString()] = false;
		}

		else if (!dictionary[lgoFoundNewParent.ToString()])
		{
			TitleElementClicked(lgoFoundNewParent, lgoFoundNewChild, false);
			dictionary[lgoFoundNewParent.ToString()] = true;
		}
	});
}
```
A simple toggle is created to handle showing an hiding child elements on click.
```csharp
void TitleElementClicked(GameObject lParent, GameObject lChild, bool lbIsActive)
{
	if (lbIsActive)
	{
		lChild.SetActive(true);
		lbIsActive = false;
	}
	else if (!lbIsActive)
	{
		lChild.SetActive(false);
		lbIsActive = true;
	}
}
```


# Technology

## Zenject Installers
`MainInstaller.cs` installs URL installer and User Interface installer. Zenject handles dependency injection in this project.
```csharp
public class MainInstaller : MonoInstaller
{
	public override void InstallBindings()
    {
		UrlInstaller.Install(Container);
		UserInterfaceInstaller.Install(Container);
	}
}
```

Bindings are separated into 2x separate installers.

```csharp
public class UrlInstaller : Installer<UrlInstaller>
{
	public override void InstallBindings()
    {
		Container.Bind<IWebRequestFactory>().To<UnityWebRequestFactory>().AsTransient();
		Container.Bind<IWebRequest>().To<UnityWebRequestWrapper>().AsTransient();

		Container.Bind<IUrlRequestBuilder>().To<UrlRequestBuilder>().FromComponentSibling().AsTransient();
		Container.Bind<IUrlManager>().To<UrlManager>().FromComponentSibling().AsSingle();
		Container.Bind<UrlManager>().FromComponentSibling();
		Container.Bind<UrlRequestBuilder>().FromComponentSibling();
		Container.Bind<UrlDeserializeResponse>().FromComponentSibling();
		Container.Bind<UrlRequestHandler>().AsSingle();
	}
}
```

```csharp
public class UserInterfaceInstaller : Installer<UserInterfaceInstaller>
{
	public override void InstallBindings()
    {
		Container.Bind<INewSearchMade>().To<NewSearchMade>().FromComponentSibling().AsSingle();

		Container.Bind<IUserInterfaceManager>().To<UserInterfaceManager>().FromComponentSibling().AsSingle();
		Container.Bind<UserInterfaceManager>().FromComponentSibling();
	}
}
```

Alongside the [Zenject](https://github.com/modesttree/Zenject) documentation, the [Ninject](http://www.ninject.org/wiki.html) documentation is useful.

## UniRx
`using UniRx;` This project features `ReactiveProperties` as observables, and basic functionality. For more information on the observer design pattern check this [video](https://www.youtube.com/watch?v=_BpmfnqjgzQ) from [Christopher Okhravi's](https://www.youtube.com/channel/UCbF-4yQQAWw-UnuCd2Azfzg) series on design patterns.


## Namespace
```csharp
// Used namespaces
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;
using UnityEngine.TestTools;
using NUnit.Framework;
using Zenject;
using NSubstitute;
using SimpleJSON;
using UniRx;
```

# Links

 - [NSubstitute](http://nsubstitute.github.io/help/getting-started/)
 - [NUnit](https://github.com/nunit/docs/wiki/Attributes)
 - [simpleJSON for Unity](http://wiki.unity3d.com/index.php/SimpleJSON)
 - [UniRx](https://assetstore.unity.com/packages/tools/unirx-reactive-extensions-for-unity-17276)
 - [Unty Test Runner](https://docs.unity3d.com/Manual/testing-editortestsrunner.html)
 - [UnityWebRequest](https://docs.unity3d.com/Manual/UnityWebRequest.html)
 - [Zenject Asset Unity Store](https://assetstore.unity.com/packages/tools/integration/zenject-dependency-injection-ioc-17758)
 - [Zenject GitHub](https://github.com/modesttree/Zenject)
