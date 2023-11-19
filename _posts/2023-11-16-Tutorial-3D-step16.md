# 3D Tutorial - Step 16 - Controller Support
by Monstrous Software


# Step 16 - Controller Support

![controller](/assets/images/controller.png)

Game controllers are, for the moment, not supported by gdx-teavm (and can actually crash the application) so this is an option that will only be available on the desktop version.
Let us add a field in Settings to enable/disable controller support:

```java
        static public boolean supportControllers = true;       // disable in case it causes issues
```
To automatically set this field we can add the following line to the `create()` method in the Main class:
```java
        Settings.supportControllers = (Gdx.app.getType() == Desktop);
```
Add the following to `GameScreen.show()`:

```java
        if (Settings.supportControllers &&  Controllers.getCurrent() != null) {
            MyControllerAdapter controllerAdapter = new MyControllerAdapter(world.getPlayerController(), this);
            Controllers.addListener(controllerAdapter);
        }
```
Create an extension of ControllerAdapter to process the inputs from the controller and 
forward them mostly to the PlayerController or sometimes to the GameScreen.
For example, the so-called DPad buttons are mapped to the WASD keys.  The left stick is used for 
player movement and the right stick for camera rotation.
This code is for an XBox compatible controller.  
There is very poor standardization for game controller buttons and sticks. If you want to support many types 
of controllers you may need to add a setup screen where the player can map their controller inputs to actions.

```java
    public class MyControllerAdapter extends ControllerAdapter {
        private final static int L2_AXIS = 4;
        private final static int R2_AXIS = 5;
    
        private final PlayerController playerController;
        private final GameScreen gameScreen;
    
        public MyControllerAdapter(PlayerController playerController, GameScreen gameScreen) {
            this.gameScreen = gameScreen;
            this.playerController = playerController;
        }
    
        @Override
        public boolean buttonDown(Controller controller, int buttonIndex) {
            processButtonEvent(controller, buttonIndex, true);
            return false;
        }
    
        @Override
        public boolean buttonUp(Controller controller, int buttonIndex) {
            processButtonEvent(controller, buttonIndex, false);
            return false;
        }
    
        private void processButtonEvent(Controller controller, int buttonIndex, boolean down) {
            // map Dpad to WASD
            if (buttonIndex == controller.getMapping().buttonDpadUp)
                buttonChange(playerController.forwardKey, down);
            if (buttonIndex == controller.getMapping().buttonDpadDown)
                buttonChange(playerController.backwardKey, down);
            if (buttonIndex == controller.getMapping().buttonDpadLeft)
                buttonChange(playerController.strafeLeftKey, down);
            if (buttonIndex == controller.getMapping().buttonDpadRight)
                buttonChange(playerController.strafeRightKey, down);
    
            if (buttonIndex == controller.getMapping().buttonR1 )
                playerController.setScopeMode(down);
            if (buttonIndex == controller.getMapping().buttonL1 )
                playerController.setRunning(down);
    
            if (buttonIndex == controller.getMapping().buttonStart && down)
                gameScreen.restart();
    
            if (buttonIndex == controller.getMapping().buttonX)
                buttonChange(playerController.switchWeaponKey, down);
            if (buttonIndex == controller.getMapping().buttonY && down)
                gameScreen.toggleViewMode();
        }
    
        private void buttonChange(int keyCode, boolean down){
            if(down)
                playerController.keyDown(keyCode);
            else
                playerController.keyUp(keyCode);
        }
 
        @Override
        public boolean axisMoved(Controller controller, int axisIndex, float value) {
            if(Math.abs(value) < 0.02f )    // dead zone to cope with neutral not being exactly zero
                value = 0;
    
            // right stick for looking
            if(axisIndex == controller.getMapping().axisRightX)     // right stick for looking around (X-axis)
                playerController.stickLookX(-value);           // rotate view left/right
            if(axisIndex == controller.getMapping().axisRightY)     // right stick for looking around (Y-axis)
                playerController.stickLookY(-value);           // rotate view up/down
    
            // left stick for moving
            if(axisIndex == controller.getMapping().axisLeftX)     // left stick for strafing (X-axis)
                playerController.stickMoveX(value);
            if(axisIndex == controller.getMapping().axisLeftY)     // right stick for forward/backwards (Y-axis)
                playerController.stickMoveY(-value);
    
            if(axisIndex == L2_AXIS)     // left trigger
               buttonChange(playerController.jumpKey, (value > 0.4f));
    
            if(axisIndex == R2_AXIS)    // right trigger
                if(value > 0.8f)
                    playerController.fireWeapon();
            return false;
        }
    }
```
In PlayerController we need to add a few methods to support this:
```java
        private final Vector2 stickMove = new Vector2();
        private final Vector2 stickLook = new Vector2();
    
    
        public void stickMoveX(float value){
            stickMove.x = value;
        }
    
        public void stickMoveY(float value){
            stickMove.y = value;
        }
    
        public void stickLookX(float value){
            stickLook.x = value;
        }
    
        public void stickLookY(float value){
            stickLook.y = value;
        }
```
And in `PlayerController.update()` we add the following lines to move the player and rotate the view 
based on controller input.  The trickiest bit is for rotating the view up and down.  We want the view
to automatically drift back to a level view when the joystick is released. In scope mode however we use the
joystick to nudge the view up or down in small steps.
```java
        // controller stick inputs
        moveForward(stickMove.y*deltaTime * moveSpeed);
        strafe(stickMove.x * deltaTime * Settings.walkSpeed);
        float delta = 0;
        float speedFactor;
        if(world.weaponState.scopeMode) {
            speedFactor = 0.2f;
            delta = (stickLook.y * 30f );
        }
        else {
            speedFactor = 1f;
            delta = (stickLook.y * 90f - stickViewAngle);
        }
        delta *= deltaTime*Settings.verticalReadjustSpeed*speedFactor;
        stickViewAngle += delta;
        rotateView(stickLook.x * deltaTime * Settings.turnSpeed*speedFactor,  delta );
```

This concludes step 16 which added game controller support to the game.