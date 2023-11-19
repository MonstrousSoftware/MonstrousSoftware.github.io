# Tutorial on creating a 3D game with LibGDX
by Monstrous Software


# Step 18 - Full Screen toggle

Let us add a feature to toggle the game between windowed mode and full screen mode using the F11 function key.
We add two class fields to remember the window size.  For the HTML version we need to make sure we catch the F11 
key rather than it being intercepted by the browser.


GameScreen:

```java
        private int windowedWidth, windowedHeight;

        @Override
        public void show() {
            ...
            Gdx.input.setCatchKey(Input.Keys.F11, true);
            ...
        }

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


This concludes step 17.