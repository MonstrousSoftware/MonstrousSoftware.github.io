## Isometric projection
How to render an isometric scene with LibGDX.

An isometric view is oftentimes used for an ‘old skool’ look as it was used in the past to fake a 3d look using only 2d sprites.  But you can also render a 3d world using an isometric view to simulate that same look.

To do this we will use the OrthographicCamera instead of the PerspectiveCamera which we more typically use for 3d scenes.

For example, imagine we have a game level of 20 by 20 tiles, centred around the origin, and we want them all in view.

If we want a top down view, we can put the orthographic camera some height above the centre and make it look down.  
Make sure the far plane distance is greater than the camera height, otherwise the whole level will be clipped and you will see nothing.
Perhaps surprisingly, the height of the camera above the ground plane is not so relevant. 
Moving the camera up and down does not have any effect because the orthographic projection does not show things bigger when they get closer.
Just make sure the camera height is between the near and far clipping distance and above any objects that you may have in your game level.

```java
	@Override
	public void create () {
		cam = new OrthographicCamera();
		cam.position.set(0f, 10f, 0f);
		cam.lookAt(0,0,0);
		cam.near = 1f;
		cam.far = 30f;
		cam.update();
		...
	}

	@Override
	public void resize (int width, int height) {
		// define view port in logical units (tiles) but maintain aspect ratio if the screen/window
		cam.viewportWidth = VIEW_SIZE;
		cam.viewportHeight = VIEW_SIZE * height/width;
		cam.update();
	}
```
Note how we pass world units (i.e. number of tiles) in resize() to set the view port size of the orthographic camera, not the number of pixels. This means we can avoid having to work with pixels per tile. We set the view port height as `VIEW_SIZE * height/width` to maintain the aspect ratio of the screen.



![image](https://user-images.githubusercontent.com/49096535/231211386-936cabf0-dcf4-43de-8443-e1fb623c8ebc.png)
> Top-down view

To show an isometric view, we place the OrthographicCamera some height above the ground plane with a view vector along on a 
diagonal of the horizontal plane and with some downward angle.  This makes a square appear like a flattened rhombus on screen.

For example let's put the camera on the diagonal X=Z, in other words we will use the same value for the camera’s X and Z position. And we choose a positive number for the camera’s Y position.  The lower the camera the more flattened the view will be. 




```java
	cam = new OrthographicCamera();
	cam.position.set(10f, 10f, 10f);
	cam.lookAt(0,0,0);
	cam.near = 1f;
	cam.far = 30f;
	cam.update();
```

The following picture shows the viewing frustum of the orthographic camera in this scene. What is inside the box of black lines, will be shown on the screen. The square at the top right is the near clipping plane and the square at the bottom right is the far clipping plane.

![image](https://user-images.githubusercontent.com/49096535/231235906-a35c3f22-0016-44a3-bf3d-488c22c18704.png)
> Projection frustum of the orthographic camera at (10,10,10) pointing at the origin

![image](https://user-images.githubusercontent.com/49096535/231210875-06458831-2074-41a5-847e-8f5f2f1bd0b7.png)
> Isometric view

Note that the isometric view can sometimes be a bit confusing because we are missing visual cues from perspective foreshortening about the distance of objects.
A cube that is a bit further away looks exactly the same as a cube hovering in the air.

![image](https://user-images.githubusercontent.com/49096535/231213400-e2016e16-e820-4fbb-afef-ea13d0f4e27c.png)
> Without shadows, the scene becomes ambiguous

It certainly helps to add shadows. And maybe you want to avoid floating cubes in your level design. 


[Source code of example code](https://github.com/MonstrousSoftware/IsoMetric)

