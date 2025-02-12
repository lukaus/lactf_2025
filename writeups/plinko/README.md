# [web/plinko](https://github.com/uclaacm/lactf-archive/tree/main/2025/web/plinko) - by chinmay

Obtain the flag by gaining 10,000 points in a web implementation of [Plinko](https://en.wikipedia.org/wiki/List_of_The_Price_Is_Right_pricing_games#Plinko)
<details> 
  <summary>Summary of my solve:</summary>
  
   Edit the physics engine to give gravity an x-component, causing the ball to fall closer to the edge, which grants more points. Bypass the server-side validation by caching the x-component of the ball's velocity. This is possible because the server validates the ball's x-velocity, but never validates the ball's x-position.
  
</details>

## The Challenge

Hint: I was tired of the rigged gambling games online, so I made this completely fair version of plinko. Don't try and cheat me.

Provided files:
- [```plinko.zip```](https://github.com/uclaacm/lactf-archive/blob/main/2025/web/plinko/plinko.zip) : zip file containing entire plinko project including the server-side source code for the plinko Docker app

Files of interest:
- [```app.js```](https://github.com/uclaacm/lactf-archive/blob/main/2025/web/plinko/app.js) - Server-side code for scoring and simulation validation
- [```public/physics.js```](https://github.com/uclaacm/lactf-archive/blob/main/2025/web/plinko/public/physics.js) : Client-side physics engine implementation


We are also given a link to plinko on the CTF's server.

## Starting Point : Exploration
I'm going to skip all the code relating to the login and account process, and instead focus only on the plinko game itself, as that is what's relevant to the method I used to solve this challenge. 

<br/><br/>
When we run the game, we see a completely fair implementation of plinko:

![Default gameplay](/media/plinko_default.gif)

<br/><br/>

Next, let's look through the code and see where the flag is referenced. 

```javascript
... // line 12 : app.js
const flag = process.env.FLAG || 'lactf{test_flag}';
```
<br/><br/>

We see it initialized from an environment variable, which means it is set when the Docker container is created, meaning it's unlikely we can get to it through file/directory manipulation. It is again referenced later in ```app.js```:
```javascript
... // line 206 : app.js
if (users[req.session['user']].points>10000) socketSend(ws, points+flag, () => ws.close());
```
So, when the user has more than 10000 points, the flag will be appended to the points value sent back to the user via the websocket. 

<br/><br/>

Here we see where the server checks for an expected physics state to make sure the client doesn't try some shenannigans. It's gonna take forever to win enough points to get the flag!
```javascript
... //line 97 : app.js
function validatePosition(prevCollision, prevVelo, prevTime, currCollision, currVelo, currTime) {
    if (typeof(prevTime)!=='number' || typeof(currTime)!=='number') return false;
    if (!prevCollision || !prevVelo || !currCollision || !currVelo) return false;
    if (!('x' in prevCollision) || !('y' in prevCollision) || !('x' in prevVelo) || !('y' in prevVelo) || !('x' in currCollision) || !('y' in currCollision) || !('x' in currVelo) || !('y' in currVelo)) return false;
    if (Math.abs(prevVelo.x-currVelo.x)>0.001) {
        return false;
    }
    const t = (currTime-prevTime);
    const posChange = calcPositionDiff(t, prevVelo.y);
    const veloChange = timeInterval*t/1000;

    const newYVelo = veloChange+prevVelo.y;
    const newYPos = posChange+prevCollision.y;

    if (Math.abs(newYVelo-currVelo.y)>0.001) {

        return false;
    }
    if (Math.abs(newYPos-currCollision.y)>0.001) {
        return false;
    }
    return true;
}
```
<br/><br/>

So let's get an idea of the overall game logic and see if there's a way to manipulate something in our favor, and win the flag.

Looking through the code, the relevant logic of the game goes as follows:
1. Client clicks to drop a ball, which creates the game board and physics engine, initializes the pegs and ball, and creates the scoring zones, then sends the ball's properties to the server : [public/physics.js, lines 45-75](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/public/physics.js#L45)
2. The server checks that the ball is starting at the correct x-position, then saves the ball's properties to validate against later. If the ball is not at x=500, the socket is closed with the message "Stop cheating."  : [app.js, lines 152-171](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/app.js#L152)
3. The client begins simulating a simple 2D frictionless kinematic physics engine : [public/physics.js, line 96](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/public/physics.js#L96)
4. If the ball collides with something, the physics engine is paused, and the ball's physics properties (location and velocity) are sent to the server, as well as the properties of whatever it collided with : [public/physics.js, line 98-125](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/public/physics.js#L98)
6. The server validates (via validatePosition()) that the ball's properties have not been modified in a way that would be inconsistent with the expected physics calculations : [app.js, lines 173-184](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/app.js#L173)
7. The server checks that the object the ball is colliding with should actually exist where the client says it is. If either of the checks fail, the socket is closed with the message "Stop cheating!!" : [app.js, lines 185-197](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/app.js#L185)
9. Otherwise, the server checks if the ball is colliding with the "ground", meaning one of the scoring zones, and sends the appropriate points (and flag if applicable) to the client before closing the socket : [app.js, lines 199-208](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/app.js#L199)
10. If the ball has collided with a peg or wall, the server will calculate how the ball should bounce, then send that back to the client : [app.js, lines 210-238](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/app.js#L210)
11. The client will recieve the message and set the ball's velocity to the resulting velocity calculated by the server, then resume physics simulation. : [public/physics.js, lines 142-147](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/public/physics.js#L142)

<br/><br/>

From here, I looked for a way to manipulate the game state to score points faster and more consistently.

## The Solve : Abusing server-side validation
My first thought was to see if I could mess with the physics engine. First, I'll use Chromium's "Overrides" feature in the Developer Tools to override ```public/physics.js``` so that I can change the client-side code as needed. Hopefully there will be some way to get the flag this way.

The physics engine used is [matter.js](https://brm.io/matter-js/), which has some useful documentation. After seeing where the physics engine was initialized on the client side ([```public/physics.js```, line 78](https://github.com/uclaacm/lactf-archive/blob/ad381b48e37a11fabc800724c50b1fdf0ddc6c80/2025/web/plinko/public/physics.js#L78)), I decided that would be as good a starting point as any. Looking at [the matter.js documentation](https://brm.io/matter-js/), I noticed that ```engine.gravity``` has an x-component! I can use that to make the ball glide gently down the side of the game board directly into the high-scoring zones. So, I change the game engine definition:
```javascript
... // line 78 : public/physics.js
        if ("error" in JSON.parse(resp)) window.location = "/login";
        //engine = Engine.create();
        engine = Engine.create({gravity:{x:0.41, y:1, scale:0.001}}); // replace the default engine gravity
        render = Render.create({
...
```
![Halfway there!](/media/plinko_halfway.gif)

After playing around with the gravity value, x=0.41 seems about right to get the ball to glide toward the edge. But, the first time the ball collides with something, including the scoring area, the socket is closed and I am told not to cheat! Oh, thats right, the server validation... Let's take a closer look.

<br/><br/>

```javascript
... // line 97 : app.js
function validatePosition(prevCollision, prevVelo, prevTime, currCollision, currVelo, currTime) {
    if (typeof(prevTime)!=='number' || typeof(currTime)!=='number') return false;
    if (!prevCollision || !prevVelo || !currCollision || !currVelo) return false;
    if (!('x' in prevCollision) || !('y' in prevCollision) || !('x' in prevVelo) || !('y' in prevVelo) || !('x' in currCollision) || !('y' in currCollision) || !('x' in currVelo) || !('y' in currVelo)) return false;
    if (Math.abs(prevVelo.x-currVelo.x)>0.001) {
        return false;
    }
    const t = (currTime-prevTime);
    const posChange = calcPositionDiff(t, prevVelo.y);
    const veloChange = timeInterval*t/1000;

    const newYVelo = veloChange+prevVelo.y;
    const newYPos = posChange+prevCollision.y;

    if (Math.abs(newYVelo-currVelo.y)>0.001) {

        return false;
    }
    if (Math.abs(newYPos-currCollision.y)>0.001) {
        return false;
    }
    return true;
}
...
```
First, the server checks that all the expected values are provided at all: the ```time``` that the current collision occurred, the ```x and y position``` of the current and last collision, and the ball's previous and current ```x and y velocity```. Then, the ```x velocity``` is checked to make sure it has not deviated much from the previous value (```x velocity``` should only really change as a result of a collision).
Then, previous collision timestamp is subtracted from the current one to find ```delta-time```, which is used via the ```calcPositionDiff()``` function to make sure the new ```y position``` makes sense in terms of gravitational velocity. If any of the values are unexpected, validation fails, otherwise ```true``` is returned. 

Notice what was never checked? ```x position```! Well, the server checks that it is provided, but the value is never checked for consistency. Still, we are headed down the right track! We just need to cache the ball's ```x velocity``` so that we can send it to the server on collision. We only need to add four more lines to public/physics.js now to solve this challenge!

<br/><br/>

First, lets just create a global variable around line 42 of ```public/physics.js``` to store ```x velocity``` in, and set it to 0.0 (since ```x velocity``` is initially 0):
```javascript
... // line 42 : public/physics.js
let prevXVel = 0.0;
...
```
<br/><br/>

We also have to set this value back to 0 every time the ball is dropped for a new game of Plinko. Otherwise, the first time the ball collides with something, the final value from the previous iteration will be sent, and the server will ~~think~~ know we are cheating. 
```javascript
... // line 78 : public/physics.js
        if ("error" in JSON.parse(resp)) window.location = "/login";
        //engine = Engine.create();
        engine = Engine.create({gravity:{x:0.41, y:1, scale:0.001}}); // replace the default engine gravity
        prevXVel = 0;
        render = Render.create({
```
<br/><br/>

Now, any time the server sends us back the result of a collision, we can save it in this variable to use later. 
```javascript
... // line 142 : public/physics.js
                        engine.timing.timestamp = collisionTime;
                        Matter.Body.setPosition(ball, collisionPos);
                        Matter.Body.setVelocity(ball, JSON.parse(resp));

                        // save the velocity
                        prevXVel = ball.velocity.x

                        engine.enabled = true;
                        Runner.run(runner, engine);
...
```
<br/><br/>

The last step is to set the ball's velocity back to this value when a collision occurs, since the server is expecting it for validation.
```javascript
... // line 104 : public/physics.js 
                    var ball = pair.bodyA.label === "ball" ? pair.bodyA : pair.bodyB;

                    // Set the ball's velocity back to the value the server will validate against:
                    ball.velocity.x = prevXVel;

                    let collisionPos = {'x': ball.position.x, 'y': ball.position.y};

                    var obs = pair.bodyA.label === "obstacle" ? pair.bodyA : pair.bodyB;
                    socket.send(JSON.stringify({"msgType": "collision", "velocity": ball.velocity, "position": ball.position, "obsPosition": obs.position, "time": engine.timing.timestamp}));
```
Sure enough, server validation is bypassed, and we can start racking up points! If the ball gets stuck, or flies off the screen, we can just refresh the page.

![Are u winnin son](/media/plinko_solve.gif)

<br/><br/>

The last step is to capture the flag as it is returned. I just put a breakpoint on line 114 so I could view the response from the server directly. Sure enough, once we have enough points, there it is:

![gottem](/media/plinko_capture.png)

This was one of the most fun challenges I was able to complete! 

While chatting with some other participants after the competition ended, it sounded like there were a lot of creative ways to solve this one. One participant said they set the ball's initial Y position much much higher than the default, which tended to cause it to bounce more aggressively toward the sides. One team removed several rows of pegs so that the ball would have a clearer shot to fall towards the high-scoring zone. Another participant said they just played it legitimately for an hour and eventually won enough points to get the flag!

This was my highest scoring challenge in the CTF. Points were granted based on total solves, so fewer solves across all teams meant more points. I was around the 75th team to solve this challenge, and only 98 teams solved it total, so I'm rather proud to have solved this one fairly easily.



