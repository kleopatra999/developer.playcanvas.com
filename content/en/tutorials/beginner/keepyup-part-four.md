---
title: Making a Simple Game - Part 4
template: tutorial-page.tmpl.html
position: 13
---

<iframe src="http://playcanv.as/p/1gDqCWa8"></iframe>

*If you haven't see [Part 1][1], [Part 2][2] and [Part 3][3] read them first.*

## The Football

The football is the center of attention in our Keepy Up game. It responds to player input, it responds to the environment (well, gravity), it makes sounds. It's probably the most complicated part of the game. Fortunately, we're going to explain all the bits to your as simply as we can.

## ball.js

```javascript
pc.script.attribute("gravity", "number", -9.8, {description: "The value of gravity to use"});
pc.script.attribute("defaultTap", "number", 5, {description: "Speed to set the ball to when it is tapped"});
pc.script.attribute("impactEffect", "entity", null, {description: "The particle effect to trigger when the ball is tapped"});
pc.script.attribute("ballMinimum", "number", -6, {description: "When ball goes below minumum y value game over is triggered"});
pc.script.attribute("speedMult", "number", 4, {description: "Multiplier to apply to X speed when tap is off center"});
pc.script.attribute("angMult", "number", -6, {description: "Multiplier to apply to angular speed when tap is off center"});

pc.script.create('ball', function (app) {
    var tmp = new pc.Vec3();

    // Creates a new Ball instance
    var Ball = function (entity) {
        this.entity = entity;
    };

    Ball.prototype = {
        // Called once after all resources are loaded and before the first update
        initialize: function () {
            this.paused = true;

            // Get the "Game" Entity and start listening for events
            this.game = app.root.findByName("Game");

            app.on("game:start", this.unpause, this);
            app.on("game:gameover", this.pause, this);
            app.on("game:reset", this.reset, this);

            // Initialize properties
            this._vel = new pc.Vec3(0, 0, 0);
            this._acc = new pc.Vec3(0, this.gravity, 0);
            this._angSpeed = 0;

            // Store the initial position and rotation for reseting
            this._origin = this.entity.getLocalPosition().clone();
            this._rotation = this.entity.getLocalRotation().clone();
        },

        // Called every frame, dt is time in seconds since last update
        update: function (dt) {
            // Don't update when paused
            if (this.paused) {
                this.entity.rotate(0, 30*dt, 0);
                return;
            }

            var p = this.entity.getLocalPosition();

            // integrate the velocity in a temporary variable
            tmp.copy(this._acc).scale(dt);
            this._vel.add(tmp);

            // integrate the position in a temporary variable
            tmp.copy(this._vel).scale(dt);
            p.add(tmp);

            // update position
            this.entity.setLocalPosition(p);

            // rotate by angular speed
            this.entity.rotate(0, 0, this._angSpeed);

            // check for game over condition
            if (p.y < this.ballMinimum) {
                this.game.script.game.gameOver();
            }
        },

        /*
         * Called by the input handler to tap the ball up in the air
         * dx is the tap distance from centre of ball in x
         * dy is the tap distance from centre of ball in y
         */
        tap: function (dx, dy) {
            // Update velocity and spin based on position of tap
            this._vel.set(this.speedMult * dx, this.defaultTap, 0);
            this._angSpeed += this.angMult * dx;

            // calculate the position of the tap in world space
            tmp.copy(this.entity.getLocalPosition());
            tmp.x -= dx;
            tmp.y -= dy;

            // trigger particle effect to tap position, facing away from the center of the ball
            this.impactEffect.setLocalPosition(tmp);
            this.impactEffect.particlesystem.reset();
            this.impactEffect.particlesystem.play();
            this.impactEffect.lookAt(this.entity.getPosition());

            // play audio
            this.entity.sound.play("bounce");

            // increment the score by 1
            this.game.script.game.addScore(1);
        },

        // Pause the ball update when not playing the game
        unpause: function () {
            this.paused = false;

            // start game with a tap
            this.tap(0, 0);
        },

        // Resume ball updating
        pause: function () {
            this.paused = true;
        },

        // Reset the ball to initial values
        reset: function () {
            this.entity.setLocalPosition(this._origin);
            this.entity.setLocalRotation(this._rotation);
            this._vel.set(0,0,0);
            this._acc.set(0, this.gravity, 0);
            this._angSpeed = 0;
        }
    };

    return Ball;
});
```

### Script Attributes

The first thing you'll notice at the top of the script are a set of script attributes that we've defined. Defining script attributes lets you expose values from your script into the editor. There are three very good reasons to do this.

First, it lets you use the same script for many different Entities with different values. For example, you could have a script attribute which sets a color, and in the editor create a red, blue and green version of a entity just by modifying the script attribute.

Second, you can quickly and easily tune the behaviour of scripts. When you modify a script attribute (or indeed any property from the editor) the changes are made instantly to any instance of the game that you have launched from the editor. So for example in the case of the `ballMinimum` property we define here, you can launch the game and test what the value of `ballMinimum` should be to allow the ball to drop off the bottom of the screen without ever having to reload the game. Test the game, modify the value, test the game.

This is known as "iteration speed". The faster you can modify and test your game, the quicker you can get it developed!

For the ball, we define script attributes that let us tweak a number of game play properties like the gravity, the impulse applied when the ball is tapped. These attributes let us very quickly tune the game to our liking.

Third, the script attribute is a great way to link a script to an Entity or an Asset in your scene. For example, the ball script needs to trigger a particle effect when it is tapped. The particle effect is on another Entity in our scene. We define a script attribute called `impactEffect` of type `entity` and in the Editor we link this to the entity with our particle effect. Our script now has a reference to the entity and we are free to modify this entity or change to another entity without breaking our code.

### The Physics Simulation

For those of you with some basic vector maths knowledge this `update()` loop of the ball should be simple, but for everyone else we'll explain a little about simulating a ball in a video game.

A simple way to simulate something in video game is to give that object an acceleration, a velocity and a position. Every time step (or frame) the acceleration (which the rate of change velocity) changes the velocity and the velocity (which is the rate of change of position) changes the position. Then you draw your object at the new position.

You can influence the position of your object in one of three ways.

* **Change the acceleration**, this is useful for applying a force over a period of time, like gravity on the ball.
* **Change the velocity**, this is an instantaneous change. Like a ball bouncing off the floor.
* **Change the position**, like teleportation, there isn't a real world equivalent!

In our simulation we have a constant acceleration due to gravity, when you tap the ball we apply an instant change in velocity and when you reset the game we teleport the ball back to it's starting position.

#### Simulating

The update loop does this:

<em>(Change in Velocity) = (Acceleration) \* (Time since last frame)</em>

<em>(New Velocity) = (Old Velocity) + (Change in Velocity)</em>

<em>(Change in Position) = (New Velocity) \* (Time since last frame)</em>

<em>(New Position) = (Old Position) + (Change in Position)</em>

In code this looks like this:

```javascript
var p = this.entity.getLocalPosition();

// integrate the velocity in a temporary variable
tmp.copy(this._acc).scale(dt);
this._vel.add(tmp);

// integrate the position in a temporary variable
tmp.copy(this._vel).scale(dt);
p.add(tmp);

// update position
this.entity.setLocalPosition(p);
```

You will note that we use temporary vector `tmp` to store intermediate values. It's important not to create a new vector every frame for this. Also notice that we have to call `setLocalPosition` to apply the updated position.

Finally, for a nice effect we add rotate the ball by the angular speed value using `entity.rotate()`. This isn't very physically accurate, but it looks nice.

#### Responding to input

You many remember from [Part 2][2] that the `input.js` script checked to see if an input has hit the ball and if so it calls the `tap()` method. The `tap()` method defined above applies a direct change to the velocity and the angular speed of the ball. We use a couple of our script attributes `this.speedMult` and `this.angMult` to multiple the new velocity and angular speed to match our expectations of the gameplay.

We also use the tap method to trigger a particle dust cloud at the point of impact and play a sound effect. We'll talk about particle and sounds in [Part 4][4].

## Summary

The ball script runs a simply physical simulation to make the ball fall under gravity and respond to taps. It also listens for game events to know when to pause and reset. Finally, it interacts with some other systems to show particle effects and play sounds.


[1]: /tutorials/beginner/keepyup-part-one
[2]: /tutorials/beginner/keepyup-part-two
[3]: /tutorials/beginner/keepyup-part-three
[4]: /tutorials/beginner/keepyup-part-four