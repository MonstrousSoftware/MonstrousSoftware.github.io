## Loading Screen



![loading](https://user-images.githubusercontent.com/49096535/230624410-08e43099-5f4f-458f-a217-7c6d83b5dc82.png)

One little annoying thing I noticed about my War of the Pixels game on HTML is how long it would take to start up from the moment the player selects "Play" in the main menu.
It can easily take up to 5 seconds. Which is not bad as loading time, but it is a long time between pressing a button and seeing anything happen. It would be better to give some visual feedback that the game is loading at that point.

The loading time is probably due to the generation of the terrain mesh from a grey scale heightfield image and from randomly placing a few hundred stones and trees on the terrain.

I could try to optimize these, but it would be a lot of work. And in any case, the loading time itself is not an issue. Just the lack of feedback the game is loading.

Another possibility is to build these things in advance. For example to construct the Terrain mesh in the Main class, not in the World class.

To explain more clearly: the main class of the game is called Main and it extends the standard Game class.  The Main class calls setScreen() to open a screen, which is extends ScreenAdapter.
And the different screens can use setScreen to switch to another screen, e.g. from the SplashScreen to the MenuScreen and from the MenuScreen to the GameScreen.

![hierarchy](https://user-images.githubusercontent.com/49096535/230628061-d45df3ce-8b31-44a8-bc1d-4184068c2627.png)

The GameScreen contains a World object which contains all the game objects, such as the tanks and airships, the terrain and the scenery (i.e. the trees and the stones).
When the player presses the "Start" button in the MenuScreen, it calls setScreen(new GameScreen()).  This constructs the GameScreen instance and calls show(). 
GameScreen.show() calls the World constructor which in turn calls the Terrain constructor and the Scenery constructor. And these last two take a few seconds to run, especially 
on the HTML/GWT platform. Only once GameScreen.show() has completed is GameScreen.resize() called, and then GameScreen.render()starts being called.

One idea to make the start button more responsive is to pre-generate the Terrain and the Scenery objects. For example, these could be created in the Main class and the World class would simply refer to the objects in the maimn class.
The idea is that the construction time is spent during game startup before the start button is even pressed.  This would work, but it is not very elegant from a code design point of view.  The main class is now contaminated with lower level classes.
Then it turns out the Scenery class needs the World object in its constructor.  So instead, we could construct the World object in the Main class. But thw World constructore needs
a camera object in its constructor.  So this is not so straightforward and quickly becomes rather hacky.

Another idea is to let GameScreen.render() show a loading screen while the world is still being constructed.  Perhaps the render function could call world.isLoaded() and show a blank screen while it is false.
However the HTML/GWT version, does not support multithreading, so this becomes a tricky proposition.

In the end, I settled for a very simple solution: to introduce a separate screen that sits between the MenuScreen and the GameScreen. I call it the PreGameScreen. The "Start" button now calls setScreen(new PreGameScreen()).
All the PreGameScreen has to do is show a "loading..." message and immediately call the GameScreen.  Then the player sees that something is happening after the button press. The loading screen stays visible during the GameScreen constructor and show() call, i.e. during construction of the World object.
It remains visibile until the very first call of GameScreen.render().

The PreGameScreen code is very simple:

```java
public class PreGameScreen extends ScreenAdapter {

    private SpriteBatch batch;
    private Main game;
    private Texture texture;
    private float timer;

    public PreGameScreen(Main game) {
        this.game = game;
    }

    @Override
    public  void show() {
        batch = new SpriteBatch();
        texture = new Texture( Gdx.files.internal("loading.png"));
        timer = 0.5f;
    }

    @Override
    public  void hide() {
        dispose();
    }
  
    @Override
    public void dispose() {
        batch.dispose();
        texture.dispose();
    }

    @Override
    public void render( float deltaTime )
    {
        timer -= deltaTime;
        if(timer < 0 ) {
            game.setScreen(new GameScreen(game));   // load game screen automatically
            return;
        }
        ScreenUtils.clear(Color.BLACK);
        batch.begin();
        batch.draw(texture, (Gdx.graphics.getWidth()-texture.getWidth())/2,(Gdx.graphics.getHeight()-texture.getHeight())/2);
        batch.end();
    }

    @Override
    public void resize(int width, int height) {
        batch.getProjectionMatrix().setToOrtho2D(0, 0, width, height);
    }

}
```

It is certainly not perfect.  An animated progress bar would be nicer.  And if the user resizes the PreGameScreen it is effectively ignored.  
But it is a simple trick to give some feedback while opening the game's main screen, which I thought I would share.
