---
layout: post
title: Card System
subtitle: Developing the core gameplay features
categories: C!YDtW
tags: [code, ui]
---

## Defining Card Data

For the actual Card Data, the obvious choice were ScriptableObjects. These are GameObjects that live outside of the scene, but can be accessed and assigned to other GameObjects. There are a couple of advantage that they offered:
- I can use them to quickly create cards without managing multiple prefabs.
- Data about a card can be read directly from the ScriptableObject.
Settled on the following code for the time being.

```c#
[System.Serializable]
public struct Resource {
    public int value;
    public ResourceType type;
}

[CreateAssetMenu(fileName = "Card", menuName = "C!YDyW/Card", order = 0)]
public class CardData : ScriptableObject {
    public Sprite image;
    public string description;
    public Resource[] cost;
    public Resource[] production;
    public int buildTime;
    public int damage;
}
```
![CardData ScriptableObject](/assets/images/2024-03-02-card-system/CardData%20ScriptableObject.png)

For now the name of the game is to give the designer as much ability to customise a card. My policy is usually give them access to as many modifiers as possible.


## Building the Card GameObject
Now that we can define what values a card is supposed to have, the second order of business is have a card show up on screen. From a single prefab, we can give the card  which then sets everything about itself from the CardData. The point for now is to get something working. As defined in the CardData class, the card is going to have at the very least:
- An Image
- Name
- Cost Value
- Production Value
Easy enough to do. Now we just need to get the CardData object attached to this Card.

![Card Prefab Properties](/assets/images/2024-03-02-card-system/Card Prefab Properties.png)

For now I just want to show something on screen, so just slapped together an Image, and a few TextMeshPro objects as children to a base image.

![Card Prefab Struture](/assets/images/2024-03-02-card-system/Card Prefab Struture.png)
```c#
public void InitCardData(CardData data) {
	this.cardData = data;

	// Retrieving the Child Objects.
	GameObject imageObject = this.transform.GetChild(0).gameObject;
	TextMeshProUGUI[] uiText = this.transform.GetComponentsInChildren<TextMeshProUGUI>();

	// Setting the reference to the UI Components on the card.
	cardImage	= imageObject.GetComponent<Image>();
	description = uiText[0];
	power		= uiText[1];
	cost		= uiText[2];

	// Setting the Card Data.
	cardImage.sprite = cardData.image;
	description.SetText(cardData.description);
	cost.SetText(cardData.cost[0].value.ToString());
	power.SetText(cardData.production[0].value.ToString());
}
```
The above code is scrappy as hell. Hardcoding and retrieving GameObject references like this but for now, we’re just worried about showing something on screen.

Once we hit play, the placeholder stuff should switch over to whatever we assigned in the CardData.
![Initialised Card](/assets/images/2024-03-02-card-system/Initialised Card.png)

For now I just want to show something on screen, so just slapped together an Image, and a few TextMeshPro objects as children to a base image.

## Interacting with the Card
Great, so now the card is taking the data we want So now let’s make it act like a card. The code used is mostly based on [Coco Code’s Tutorial](https://youtu.be/kWRyZ3hb1Vc) for a drag and drop system, with some extra logic added on top. 

But the short version is that dragging around an object is actually really straightforward with Unity’s `IBeginDragHandler`, `IDragHandler`, `IEndDragHandler` interfaces. Each interface method takes a PointerEventData, which is just a reference to whatever is triggering this event (i.e. the mouse). This is all the code you need to drag around the card.

```c#
public void OnDrag(PointerEventData eventData) {
	this.transform.position = Input.mousePosition;
}
```
If we just wanted to drag around something like a pop up, this would be enough. But to be able to snap cards back to the hand, or into the play area, we need a few extra steps.
```c#
public Image baseImage;
public Transform parentAfterDrag; // We need this to be public later.

public void OnBeginDrag(PointerEventData eventData) {
	// Store what it was parented to to reparent later.
	parentAfterDrag = this.transform.parent;
	this.transform.SetParent(this.transform.root);
	// Moves the object to the bottom of the heirarchy which will render it on top of everything else.
	this.transform.SetAsLastSibling();
	baseImage.raycastTarget = false;
}

public void OnEndDrag(PointerEventData eventData) {
	// It will either reparent to the player hand
	// Or the CardSlot will set itself as the parent.
	transform.SetParent(parentAfterDrag);
	baseImage.raycastTarget = true;
}
```
But what if we want to make other things in the game draggable that aren’t a card? Lets move all the interaction logic to a DraggableObject Class. This will let us attach this class to anything that we might want to drag around in the game. This code is also unlikely to change, so this also lets us remove a small chunk of code Card class.

Within the Card Class, we then include any code that only has logic pertaining to a card.

## Spawning Cards Dynamically
When we draw a card, ideally we only keep a single Card prefab to serve as the template, and work with CardData.

![Draw Card Button](/assets/images/2024-03-02-card-system/Draw Card Button.png)
```c#
public void DrawCard() {
	// Get a random card from the available cards.
	CardData cardToPlay = inDeck[Random.Range(0, inDeck.Count())];
	GameObject newCardObject = Instantiate(cardPrefab);
	Card newCard = newCardObject.GetComponent<Card>();
	newCard.InitCardData(cardToPlay);
	// Attach the card to the player hand.
	newCardObject.transform.SetParent(playerHandArea.transform);
	inHand.Add(newCard);
}
```

## Placing the Cards in Play
Now that the card can be moved around, we can focus on getting it in play. 

pointerDrag retrieves the GameObject that is currently being dragged. From here, we can access the instance of the Card class attached to it.

```c#
public event Action<Card> OnCardPlayed;

public void OnDrop(PointerEventData eventData) {
	Card card = eventData.pointerDrag.GetComponent<Card>();
	// Only set this as a parent if there isn't a card already attached to the slot.
	if(this.transform.childCount < 1) {
			card.parentAfterDrag = this.transform;
			OnCardPlayed?.Invoke(card);
	}
}
```

We can then add a method in the GameManager to handle all of the high level logic, such as deducting resources, telling the UIManager to update the UI, etc etc.

This works great, but what if I want to add or remove card slots? Sure, we can just duplicate the GameObject, but what about at Runtime? UnityEvents are great, but they’re a bit of a pain to dynamically subscribe to and manage in code.

So, we need to take a step backwards to regular old C# Events. They’re managed within scripts and are not as friendly to a designer, since if we want to subscribe to said event, we need to do it in code. But, realistically, for now, I’d rather give the designer the flexibility of dynamically add CardSlots. Besides, the only thing the CardSlot should be doing is, once again, telling the Game Ma

Why isn’t it working?

Note where the mouse cursor is, remember we turned off raycasting for the card background? It would probably be a good idea to turn these off the assets by default for the same reason.

![Placed Cards](/assets/images/2024-03-02-card-system/Placed Cards.png)

Ok great, so now we have a card that is moving around, and can be placed in it’s spot. Now what?

## Calling the Game Manager
Ideally, the only thing a GameObject should be handling is it’s own state and data. The CardSlot or the Card shouldn’t be calling methods that can alter the game state or UI directly. It just becomes a nightmare to maintain, having to look through different scripts if we want to update/fix something in the UI. Instead, I want to hand this off to a single script that will handle the high level game logic for us. This is where an embarrassingly recent discovery comes in: Unity Events.

Anyway, in case you need a refresher, an Event is a fancy type of function that allows other functions (known as Delegates) to subscribe to it. At some point in our code, we can invoke the Event, which in turn, tells C# to call any Delegate methods subscribed to the Event. This lets us avoid checking something every frame in the Update method wasting valuable resources.
```c#
public void CardPlayed(Card card) {
	inHand.Remove(card);
	inPlay.Add(card);
	gameState.doomMeter++;
	foreach(Resource cost in card.cardData.cost) {
		gameState.resources[cost.type] -= cost.value;
	}
	updateUI.Invoke();
}
```

## Updating the UI
Something I thought might be cool is having cards could potentially have multiple resources assigned to the cost and production. I wanted something similar to the way resources are shown in 7 Wonders, with different resource types being stacked on top of each other.


![First Draft of UI](/assets/images/2024-03-02-card-system/First Draft of UI.png)