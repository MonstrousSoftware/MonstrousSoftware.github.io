# 3D Tutorial - Step 18 - Full Screen toggle
by Monstrous Software


# Step 18 - Full Screen toggle

Let us add a feature to toggle the game between windowed mode and full screen mode using the F11 function key.
We add two class fields to remember the window size.  


GameScreen:

```java
        private int windowedWidth, windowedHeight;

        private void toggleFullScreen() {        // toggle full screen / windowed screen
            if (!Gdx.graphics.isFullscreen()) {
                windowedWidth = Gdx.graphics.getWidth();        // remember current width & height
                windowedHeight = Gdx.graphics.getHeight();
                Gdx.graphics.setFullscreenMode(Gdx.graphics.getDisplayMode());
                resize(Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
            } else {
                Gdx.graphics.setWindowedMode(windowedWidth, windowedHeight);
                resize(windowedWidth, windowedHeight);
            }
        }

        @Override
        public void render(float delta){
            ...
            if(Gdx.input.isKeyJustPressed(Input.Keys.F11))
                toggleFullScreen();
            ...
        }
```


This concludes step 18.

This concludes the tutorial, although there are a lot of things to improve and to polish on the game, the principles of coding a basic 3d game should now hopefully be clear.
Thank you for reading up to here and if you intend to apply some of these lessons I wish you the best of luck.