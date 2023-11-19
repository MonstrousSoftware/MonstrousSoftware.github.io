# Tutorial on creating a 3D game with LibGDX
by Monstrous Software


# Step 11 - Asset Manager and Sounds

Let us add some sound effect to the game, but before we do that we should load all the game assets via the asset manager.  We will 
encapsulate LibGDX's AssetManager in an Assets class which will be a field of the Main class.

## Asset Manager

Create a new Assets class to load the GLTF file and some sound effect files.  The Asset class also defined constants for the different sound effects and
some public field for commonly used assets, such as skin and font that we will use in the next step for the User Interface.

```java
        public class Assets implements Disposable {
        
            public class AssetSounds {
        
                // constants for sounds in game
                public final Sound COIN;
                public final Sound FALL;
                public final Sound GAME_OVER;
                public final Sound HIT;
                public final Sound JUMP;
                public final Sound GAME_COMPLETED;
                public final Sound UPGRADE;
                public final Sound GUN_SHOT;
        
                public AssetSounds() {
                    COIN = assets.get("sound/coin1.ogg");
                    FALL = assets.get ("sound/fall1.ogg");
                    GAME_OVER = assets.get ("sound/gameover1.ogg");
                    HIT  = assets.get("sound/hit1.ogg");
                    JUMP  = assets.get("sound/jump1.ogg");
                    GAME_COMPLETED  = assets.get("sound/secret1.ogg");
                    UPGRADE = assets.get ("sound/upgrade1.ogg");
                    GUN_SHOT = assets.get ("sound/9mm-pistol-shoot-short-reverb-7152.mp3");
                }
            }
        
            public AssetSounds sounds;
            public Skin skin;
            public BitmapFont uiFont;
            public SceneAsset sceneAsset;
            public Texture scopeImage;
        
            private AssetManager assets;
        
            public Assets() {
                Gdx.app.log("Assets constructor", "");
                assets = new AssetManager();
        
                assets.load("ui/uiskin.json", Skin.class);
        
                assets.load("font/Amble-Regular-26.fnt", BitmapFont.class);
        
                assets.setLoader(SceneAsset.class, ".gltf", new GLTFAssetLoader());
                assets.load( Settings.GLTF_FILE, SceneAsset.class);
        
                assets.load("sound/coin1.ogg", Sound.class);
                assets.load("sound/fall1.ogg", Sound.class);
                assets.load("sound/gameover1.ogg", Sound.class);
                assets.load("sound/hit1.ogg", Sound.class);
                assets.load("sound/jump1.ogg", Sound.class);
                assets.load("sound/secret1.ogg", Sound.class);
                assets.load("sound/upgrade1.ogg", Sound.class);
                assets.load("sound/9mm-pistol-shoot-short-reverb-7152.mp3", Sound.class);
        
        
                assets.load("images/scope.png", Texture.class);
            }
        
            public void finishLoading() {
                assets.finishLoading();
                initConstants();
            }
        
            private void initConstants() {
                sounds = new AssetSounds();
                skin = assets.get("ui/uiskin.json");
                uiFont = assets.get("font/Amble-Regular-26.fnt");
                sceneAsset = assets.get(Settings.GLTF_FILE);
                scopeImage = assets.get("images/scope.png");
            }
        
            public <T> T get(String name ) {
                return assets.get(name);
            }
        
            @Override
            public void dispose() {
                Gdx.app.log("Assets dispose()", "");
                assets.dispose();
                assets = null;
            }
        }
```

We will create the Assets object as a public static field of the Main class.  This allows the assets to be used from any screen in the game, for 
example, from menu screens as well as the game screen.  If there are a lot of assets, it is best practice to use Asset Manager's asynchronous loading
perhaps while displaying a load screen and a progress bar to the user.  This avoids that the game would become unresponsive during start up.
As we have only a few assets, we will load them all synchronously by calling `finishLoading()` immediately.

```java
        public class Main extends Game {

            public static Assets assets;
        
                @Override
                public void create() {
                    Gdx.app.log("Main", "create()");
                    assets = new Assets();
                    assets.finishLoading();
                    setScreen(new GameScreen(this));
                }
            
                @Override
                public void dispose() {
                    Gdx.app.log("Main", "dispose()");
                    assets.dispose();
                    super.dispose();
                }
            }
        }
```

Now we can update the World class to get the sceneAsset field from the Assets class.
Note that since we load the asset from the asset manager we should NOT dispose sceneAsset anymore in the `World.dispose()` method, this will be done
in Assets.dispose().

```java
    public World() {                // <---- removed parameter
        gameObjects = new Array<>();
        stats = new GameStats();
        sceneAsset = Main.assets.sceneAsset;                // <----- use Main.assets
        isDirty = true;
        physicsWorld = new PhysicsWorld(this);
        factory = new PhysicsBodyFactory(physicsWorld);
        rayCaster = new PhysicsRayCaster(physicsWorld);
        playerController = new PlayerController(rayCaster);
    }
    ...
    @Override
    public void dispose() {
        physicsWorld.dispose();                     // <--- removed sceneAsset.dispose() !
        rayCaster.dispose();
    }
```

## Sound effects

Now we can add sound effects for example in the World class when we pick up a coin or a health pack in `World.pickup()`:

```java
        private void pickup(GameObject character, GameObject pickup){
            if(pickup.type == GameObjectType.TYPE_PICKUP_COIN) {
                inventory.coinsCollected++;
                Main.assets.sounds.COIN.play();
            }
            else if(pickup.type == GameObjectType.TYPE_PICKUP_HEALTH) {
                character.health = Math.min(character.health + 0.5f, 1f);
                Main.assets.sounds.UPGRADE.play();
            }
            removeObject(pickup);
        }
```

And sound effects for bullet impact and player death:

```java
        private void bulletHit(GameObject character, GameObject bullet) {
            removeObject(bullet);
            character.health -= 0.25f;      // - 25% health
            Main.assets.sounds.HIT.play();
            if(character.isDead()) {
                removeObject(character);
                if (character.type.isPlayer)
                    Main.assets.sounds.GAME_OVER.play();
            }
        }
```

We can also play a little sound effect, also in the World class, when the game is completed, i.e. all coins are collected and all enemies are gone.

```java
        public void update( float deltaTime ) {

            if(stats.numEnemies > 0 || stats.coinsCollected < stats.numCoins)
                stats.gameTime += deltaTime;
            else {
                if(!stats.levelComplete)
                    Main.assets.sounds.GAME_COMPLETED.play();
                stats.levelComplete = true;
            }
            playerController.update(player, deltaTime);
            physicsWorld.update();
            syncToPhysics();
            for(GameObject go : gameObjects)
                go.update(this, deltaTime);
        }
```

This concludes step 11 having introduced the asset manager and sound effects.