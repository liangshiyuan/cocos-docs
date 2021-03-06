# EventManager Mechanism

* version: since cocos2d-html5 v3.0 alpha0

## Introduction

Cocos2d-html5 3.0 introduces a new mechanism for responding to user events. This document explains how it works.

The basics:

- **Event listeners** encapsulate your event processing code.
- **Event Manager** manage listeners of user events.
- **Event objects** contain information about the event.

To respond to events, you must first create an **cc.EventListener**. There are five different kinds of EventListeners:

* `cc.EventListenerTouch` - responds to touch events
* `cc.EventListenerKeyboard` - responds to keyboard events
* `cc.EventListenerAcceleration` - reponds to accelerometer events
* `cc.EventListenMouse` - responds to mouse events
* `cc.EventListenerCustom` - responds to custom events

Then, attach your event processing code to the appropriate callback on the event listener (e.g., `onTouchBegan` for `EventListenerTouch` listeners, or `onKeyPressed` for keyboard event listeners).

Next, register your EventListener with the **cc.eventManager**.

When events occur (for example, the user touches the screen, or types on the keyboard), the `cc.eventManager` distributes **Event objects** (e.g. `EventTouch`, `EventKeyboard`) to the appropriate EventListeners by calling your callbacks. Each Event object contains information about the event (for example, the coordinates where the touch occurred).


## Example

In the following example, we will add three overlapping buttons to a scene. Each will respond to touch events.

### Create three sprites with images for the buttons

```javascript

	var sprite1 = cc.Sprite.create("Images/CyanSquare.png");
	sprite1.x = size.width/2 - 80;
	sprite1.y = size.height/2 + 80;
    this.addChild(sprite1, 10);
    
    var sprite2 = cc.Sprite.create("Images/MagentaSquare.png");
	sprite2.x = size.width/2;
	sprite2.y = size.height/2;
    this.addChild(sprite2, 20);
    
    var sprite3 = cc.Sprite.create("Images/YellowSquare.png");
	sprite3.x = 0;
	sprite3.y = 0;
    sprite2.addChild(sprite3, 1);	
```
                                                
![touchable_sprite](res/touchable_sprite.png)

### Create a single touch event listener and write the callback code
```javascript

	//Create a "one by one" touch event listener (processes one touch at a time)
    var listener1 = cc.EventListener.create({
	    event: cc.EventListener.TOUCH_ONE_BY_ONE,
		// When "swallow touches" is true, then returning 'true' from the onTouchBegan method will "swallow" the touch event, preventing other listeners from using it.
	    swallowTouches: true,
		//onTouchBegan event callback function						
	    onTouchBegan: function (touch, event) {	
			// event.getCurrentTarget() returns the *listener's* sceneGraphPriority node.	
		    var target = event.getCurrentTarget();	
		    
			//Get the position of the current point relative to the button
		    var locationInNode = target.convertToNodeSpace(touch.getLocation());	
		    var s = target.getContentSize();
		    var rect = cc.rect(0, 0, s.width, s.height);
		    
			//Check the click area
		    if (cc.rectContainsPoint(rect, locationInNode)) {		
			    cc.log("sprite began... x = " + locationInNode.x + ", y = " + locationInNode.y);
			    target.opacity = 180;
			    return true;
		    }
		    return false;
	    },
		//Trigger when moving touch
	    onTouchMoved: function (touch, event) {			
		    //Move the position of current button sprite
			var target = event.getCurrentTarget();
		    var delta = touch.getDelta();
		    target.x += delta.x;
		    target.y += delta.y;
	    },
		//Process the touch end event
	    onTouchEnded: function (touch, event) {			
		    var target = event.getCurrentTarget();
		    cc.log("sprite onTouchesEnded.. ");
		    target.setOpacity(255);
			//Reset zOrder and the display sequence will change
		    if (target == sprite2) {					
		    	sprite1.setLocalZOrder(100);
		    } else if (target == sprite1) {
		    	sprite1.setLocalZOrder(0);
		    }
	    }
    });
	
```
**cc.EventListener.create** is a creator of all type of event listeners,  parameter 'event' can be used to set event listener's type. for example 'cc.EventListener.TOUCH_ONE_BY_ONE' and cc.EventListenerTouchOneByOne .

### Add event listener to event dispatcher

```javascript

	// add listeners to cc.eventManager
	cc.eventManager.addListener(listener1, sprite1);
	cc.eventManager.addListener(listener1.clone(), sprite2);
	cc.eventManager.addListener(listener1.clone(), sprite3);
```

**cc.eventManager** is a singleton object in Cocos2d-html5. You can use it to manage the dispatching state of all events. You can add listener to cc.eventManager through `addListener`,  `addListener` supports two parameters: `listener` and `nodeOrPriority`, if `nodeOrPriority` is a cc.Node or cc.Node's subclass object, it will add `listener` as SceneGraphPriority, and if `nodeOrPriority` is number value, it will add `listener` as FixedPriority. 

*Note:* In the example above, we use the `clone()` method in the second and third calls to `addListener`. This is because each event listener can be added only once. The methods `addListener` set a registration flag on the event listener, and won't add the event listener again if the flag is already set.

One more thing to keep in mind: if you add a _fixed priority_ listener to a node, you need to remove the listener manually when the node is removed. However, in the case of a _scene graph priority_ listener, the listener is bound to the node: when the node's destructor is called, the listener will be removed automatically.

### A faster way to add listener to cc.eventManager
```javascript

    cc.eventManager.addListener({
	    event: cc.EventListener.TOUCH_ALL_AT_ONCE,
	    onTouchesMoved: function (touches, event) {
		    var touch = touches[0];
		    var delta = touch.getDelta();
		    
		    var node = event.getCurrentTarget().getChildByTag(TAG_TILE_MAP);
		    var diff = cc.pAdd(delta, node.getPosition());
		    node.setPosition(diff);
	    }
    }, this);
```
**cc.eventManager**的 `addListener`'s first parameter `listener` supports two types: cc.EventListener object and json object.  if `listener` is a json object, it will create a cc.EventListener object by json object.

### New touch mechanism

The above procedures may seem more difficult compared to the event mechanism in version 2.x. In the old version, you would inherit from a delegate class, which defined `onTouchBegan()` methods, etc. Your event handling code would go into these delegate methods.

This new event mechanism removes the event processing logic from the delegate and encapsulates it into a listener. However, the logic above implements the functions below:

1. By using an event listener, a sprite can be added to the event manager with *SceneGraphPriority*. That is, when clicking a sprite button, callback functions will be called in the same order they are drawn in (that is, sprites that are in front of other sprites will get the touch events first).
2. When dealing with event logical, according to each kind of situations to solve the logics when touched(such as recognize click area, set clicked element as different transparency) to display the click effect.
3. As `swallowTouches: true,` is setted and some decisions has been made in onTouchBegan to get the return value, whether the display order of the touch event should pass back can be solved.   

### FixedPriority vs SceneGraphPriority

The eventManager uses priorities to decide which listeners get delivered an event first.

- **FixedPriority** is an integer value. Event listeners with lower `Priority` values get to process events before event listeners with higher `Priority` values.
- **SceneGraphPriority** is a pointer to a `Node`. Event listeners whose Nodes have higher Z order values (that is, are drawn on top) receive events before event listeners whose Nodes have lower Z order values (that is, are drawn below). This ensures that touch events, for example, get delivered front-to-back, as you would expect.

##Other event dispatch handling modules

In addition to touch event response, the following modules using the same event handling approach.

###Keyboard events

In addition to keyboard, it also can be every menu of the device, they can use the same listener to deal with the event.

```javascript

	//add a keyboard event listener to statusLabel
	cc.eventManager.addListener({
	    event: cc.EventListener.KEYBOARD,
	    onKeyPressed:  function(keyCode, event){
		    var label = event.getCurrentTarget();
		    label.setString("Key " + keyCode.toString() + " was pressed!");
	    },
	    onKeyReleased: function(keyCode, event){
		    var label = event.getCurrentTarget();
		    label.setString("Key " + keyCode.toString() + " was released!");
	    }
    }, statusLabel);    
``` 
### Accelerometer events

Before using accelerometer events, you need to enable them on the cc.inputManager:

`cc.inputManager.setAccelerometerEnabled(true);`

Then create the corrresponding listener. 

```javascript

        cc.eventManager.addListener({
            event: cc.EventListener.ACCELERATION,
            callback: function(acc, event){
 				//  Processing logic here 
            }
        }, sprite);	
```
###Mouse events

Mouse click event dispatching and touch event dispatching have been managed by cc.eventManager in version 3.0. It works multi-platform, and enrich the user experience of the game.
```javascript

    cc.eventManager.addListener({
	    event: cc.EventListener.MOUSE,
	    onMouseMove: function(event){
		    var str = "MousePosition X: " + event.getCursorX() + "  Y:" + event.getCursorY();
		    // do something...
	    },
	    onMouseUp: function(event){
		    var str = "Mouse Up detected, Key: " + event.getButton();
		    // do something...
	    },
	    onMouseDown: function(event){
		    var str = "Mouse Down detected, Key: " + event.getButton();
		    // do something...
	    },
	    onMouseScroll: function(event){
		    var str = "Mouse Scroll detected, X: " + event.getCursorX() + "  Y:" + event.getCursorY();
		    // do something...
	    }
    },this);

```

###Custom Event

The event types above are defined by the system, and the events (such as touch screen, keyboard response etc) are triggered by the system automatically. In addition, you can make your own _custom events_ which are not triggered by the system, but by your code, as follows:

```javascript

    var _listener1 = cc.EventListener.create({
	    event: cc.EventListener.CUSTOM,
	    eventName: "game_custom_event1",
	    callback: function(event){
	    	statusLabel.setString("Custom event 1 received, " + event.getUserData() + " times");
	    }
    });    
    cc.eventManager.addListener(this._listener1, 1);
```

A custom event listener has been defined above, with a response method, and added to the event dispatcher. So how is the event handler triggered? Check it out:

```javascript
	
    ++selfPointer._item1Count;
    var event = new cc.EventCustom("game_custom_event1");
    event.setUserData(selfPointer._item1Count.toString());
    cc.eventManager.dispatchEvent(event);		
```

The above example creates an `EventCustom` object and sets its UserData. It is then dispatched manually with `cc.eventManager.dispatchEvent(event);`. This triggers the event handler defined previously.

###Removing event listeners

An added listener can be removed with following method:

```javascript

	cc.eventManager.removeListener(listener);			//remove a listener from cc.eventManager
	cc.eventManager.removeListener(aSprite);			//remove all listeners of aSprite from cc.eventManager
```

To remove all the listeners of the cc.eventManager, use the following code:

```javascript
    cc.eventManager.removeAllListeners();

```

When using `removeAll`, all the listeners of this node can be removed. Removing specific listener is a recommending way. 

Note: After using `removeAll` menu can not respond, because it also needs accepting touch event.
