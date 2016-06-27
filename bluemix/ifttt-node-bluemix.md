# Building a home automation solution with Hue, IFTTT and Node-RED on Bluemix in 15 Minutes

Philips Hue is a lightning system which is enabled to be used as an IoT device. This tutorial explains how the state of the Philips Hue lights is controlled by complex rules based on location data received from a mobile device and sunrise/ sunset information from a weather service. You will learn how to build an easy extendable home automation solution in just a view minutes leveraging free available tools like IFTTT, Node-RED and IBM Bluemix.

---

![Solution Overview](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-06-26_iot_overview.png)

The goal of the home automation solution is to turn on/ off the lights based on the following rules: The lights are turned on if one of the following situations is true:

* It is dark outside (between sunset and sunrise) *AND* the resident is not at home *AND* it's before 11pm
* It is dark outside *AND* the resident is at home (and has not turned of the lights manually)

This goal will be achieved by utilizing channels available on IFTTT to control the Hue lights, gathering location data from the resident's mobile device as well as weather data. Since IFTTT is not able to realize complex logical relations between channels it will be connected via the IFTTT Maker channel to a Node-RED flow running on Bluemix which implements the rules defined above.

## Prerequisites

To follow the steps below one should have:

* An IFTTT account
* Philips Hue lights connected to IFTTT
* A mobile iOS device connected to IFTTT
* An IBM Bluemix account

## Step 1: Create Node-RED app on IBM Bluemix

The first step is to create a Node-RED application on Bluemix. Detailed instructions for this can be found [here](https://developer.ibm.com/recipes/tutorials/creating-a-nodered-application-on-bluemix/).

When the project is created two additional dependencies (`"node-red-contrib-react": " ~0.0.1"`, `"lodash": "~4.13.1"`) are added to `package.json`. To make them available in Node-RED's global context `bluemix-settings.js` file needs to be edited:

```javascript
{
  // ...
  functionGlobalContext: {
    "lodash": require("lodash"),
    "ReactRED": require("node-red-contrib-react")
  }
  // ...
}
```

Make sure that the application is redeployed before implementing the flow in Step 3. Editing and deploying the files can be done within the browser with [Bluemix DevOps Services](https://hub.jazz.net/docs/edit/).

## Step 2: Create IFTTT recipes

IFTTT can be seen as the inbound and outbound gateway as it translates events from devices and services into HTTP calls to trigger the logic running on Node-RED and vice-versa.

The following IFTTT channels are used:

* [Weather Channel](https://ifttt.com/weather) to receive events on sunrise and sunset
* [iOS Location Channel](https://ifttt.com/ios_location) to receive events when entering and leaving the home area
* [Maker Channel](https://ifttt.com/maker) to send and receive events to/ from the Node-RED app on Bluemix
* [Philips Hue Channel](https://ifttt.com/hue) to control the lights


To receive events from the devices the four recipes need to be created:

* IF `Sunrise` (Weather Channel) THEN `make a web request to http://${bluemix-application-url}/ifttt-trigger?event=sunrise` (Maker Channel)
* IF `Sunset` (Weather Channel) THEN `make a web request to http://${bluemix-application-url}/ifttt-trigger?event=sunset` (Maker Channel)
* IF `You enter an area` (iOS Location Channel) THEN `make a web request to http://${bluemix-application-url}/ifttt-trigger?event=arriveAtHome` (Maker Channel)
* IF `You exit an area` (iOS Location Channel) THEN `make a web request to http://${bluemix-application-url}/ifttt-trigger?event=leaveHome` (Maker

![Channel Creation](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-06-26_iot_channels_1.gif)

To control the Hue lights it's two additional recipes are created:

* IF `Maker event 'lights_true'` (Maker Channel) THEN `turn on Hue lights` (Hue Channel).
* IF `Maker event 'lights_false'` (Maker Channel) THEN `turn off Hue lights` (Hue Channel).

![Channel Creation](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-06-26_iot_channels_2.gif)

For details how recipes are created see [here](http://www.makeuseof.com/tag/how-to-create-your-own-ifttt-recipes-for-automating-your-favorite-sites-and-feeds/).

## Step 3: Create Node-RED flow

The final step is to create the Node-RED flow which will integrate our devices withe the logic which implements the rules defined at the top. The flow uses 6 nodes.

![Node-RED flows](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-06-26_iot_flows.png)

* `IFTTT` http input node: Receives the Maker requests from IFTTT on Url `/ifttt-trigger`
* `Ok Response` http output node: Responds the message with HTTP Status Code 200.
* `Timer` inject node: Triggers an event every 5 minutes.
* `ITTT Maker` http request node: Calls the Maker urls on IFTTT `https://maker.ifttt.com/trigger/{{{payload.triggerWithValue}}}/with/key/${maker-key}`

The first function node `prepare payload` transforms the message to a timer event which is similar as the input of the Maker calls which will be send from the http input node:

```javascript
var _ = context.global.get('lodash');
var date = new Date(msg.payload);
var hours = date.getHours();
var minutes = "0" + date.getMinutes();
var formattedTime = hours * 100 + minutes * 1;

return _.assign({}, msg, {
  payload: {
    event: 'timerEvent',
    value: formattedTime
  }
});
```

Within the `rules` node the rules are defined with help of `node-red-contrib-react` (For detailed information visit [GitHub](https://github.com/cokeSchlumpf/node-red-contrib-react)):

```javascript
var
    ReactRED = context.global.get('ReactRED'),
    Props = ReactRED.Props,
    Thing = ReactRED.Thing,
    bindEvent = ReactRED.bindEvent;

context.global.set('home', context.global.get('home') || Thing({
  _stateTypes: {
    currentTime: bindEvent(
      Props.number().withDefault(0), [
        [ "timerEvent", "value", true ]
      ]),

    sunIsShining: bindEvent(
      Props.bool().withDefault(true), [
        [ "sunrise", true ],
        [ "sunset", false ]
      ]),

    somebodyAtHome: bindEvent(
      Props.bool().withDefault(false), [
        [ "arriveAtHome", true ],
        [ "leaveHome", false ]
      ]),

    lights: Props.bool().withDefault(false)
  },

  _rules: function() {
    return {
      lights: !this.sunIsShining && (this.somebodyAdHome || this.currentTime < 2359)
    };
  }
}));

return [ context.global.get('home').handle(msg) ];
```

That's it. After deploying the flow to node red the Hue lights are turned on and off based on the defined rules.

## Conclusion

This short example shows how easy one can connect several devices via standardized services. Leveraging these services combined with some smart libraries makes it very simple to implement complex relationships between the state of several devices.

The shown solution can be easily extended by additional devices and events from other services. Since node-red-contrib-react hides the complex logic to react to events in the right way rules can be changed and extended in just a few seconds.
