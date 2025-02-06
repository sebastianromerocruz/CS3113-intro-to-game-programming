<h2 align=center>Week 03</h2>

<h1 align=center>Delta Time</h1>

<h3 align=center>7 Pegasus Moon, Imperial Year MMXXV</h3>

***Song of the day***: _[**Why Can’t You Love Me? [Band Ver]**](https://www.youtube.com/watch?v=33tITqSwGB8) by Wendy (2021)._

---

### Sections

1. [**Matrix Operations Review**](#mat)
2. [**Spaces**](#spaces)
3. [**Timing, FPS, and Delta Time**](#delta)

---

<a id="mat"></a>

### Part 1: _Matrix Operations Review_ 

Recall our three transformations, with their respective matrices:

- **Scaling**:

![scal-mult-ex](assets/scal-mult-ex.svg)

<sub>**Figure 1**: A matrix being scaled by a scalar value of 2.</sub>

- **Rotation**:

![scal-scal](assets/rot-mat-ex.svg)

<sub>**Figure 2**: A column vector being rotated by an angle of Θ.</sub>

- **Translation**:

![scal-trans](assets/translation-mat.svg)

<sub>**Figure 3**: The standard transformation matrix for the x- and y-dimensions.</sub>

Thus far, we have been applying these transformations to the model matrix one after the other:

```c++
g_model_matrix = glm::translate(g_model_matrix, glm::vec3(TRAN_VALUE, 0.0f, 0.0f));
g_model_matrix = glm::scale(g_model_matrix, scale_vector);
g_model_matrix = glm::rotate(g_model_matrix, ROT_ANGLE, glm::vec3(0.0f, 0.0f, 1.0f));
```

Mathematically speaking, there's nothing inherently wrong with doing this. Ideally, though, we would like to minimize the number of times our model matrix is modified each frame. Instead of modifying it directly multiple times, a better approach is to store transformation parameters separately—such as position, scale, and rotation—and tyhen construct the model matrix from scratch each frame.

This approach is crucial for keeping transformations independent and preventing cumulative errors. If we were to continuously modify the same model matrix without resetting it, transformations would compound over time, leading to unintended effects like exponential scaling, drifting positions, or erratic rotations. To prevent this, we reset the model matrix to the identity matrix (`glm::mat4(1.0f)`) every frame before applying the stored transformations. This ensures that each transformation is applied fresh and in the correct order, rather than accumulating inconsistencies over multiple frames. Separating transformations also makes debugging easier, as each component can be adjusted individually without affecting the others in unexpected ways.

#### OpenGL Example: Applying Transformations Efficiently

To illustrate this in OpenGL, consider how we might set up transformations for an object in a render loop:

```c++
void update()
{
    glm::mat4 g_model_matrix = glm::mat4(1.0f); // Reset the model matrix

    // Apply transformations using separate variables
    g_model_matrix = glm::translate(g_model_matrix, object_position);
    g_model_matrix = glm::rotate(g_model_matrix, glm::radians(object_rotation), glm::vec3(0.0f, 0.0f, 1.0f));
    g_model_matrix = glm::scale(g_model_matrix, object_scale);
}
```

Here, `object_position`, `object_rotation`, and `object_scale` are stored separately and used to construct the model matrix each frame. This avoids cumulative errors and makes it easy to update any transformation independently.

#### Incorrect Way: Accumulating Transformations

Contrast this with an approach where transformations are accumulated over frames:

```c++
void update()
{
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(0.01f, 0.0f, 0.0f)); // Continuously modifying the same matrix
    g_model_matrix = glm::rotate(g_model_matrix, glm::radians(1.0f), glm::vec3(0.0f, 0.0f, 1.0f)); // Adding rotation every frame
}
```

This approach will cause the transformations to accumulate indefinitely, leading to drifting positions and unexpected rotations over time.

By keeping transformations separate and resetting the model matrix each frame, we ensure predictable and stable rendering behavior.

<a id="spaces"></a>

<br>

### Part 2: _Spaces_

This doesn't only apply between models and the world—it also happens between models. Let's say that our square is the main character of our game, and straight ahead lies an item:

```


  +---+                  X
  |   |                  |
  +---+                  |
—————————————————————————————————————
 Character              Item
```

<sub>**Figure 8**: Our character facing an item.</sub>

If we were to translate both items _with respect to the world_, the code would be similar to what we've seen before:

```c++
g_character_model_matrix = glm::mat4(1.0f);
g_character_model_matrix = glm::translate(g_character_model_matrix, glm::vec3(TRAN_VALUE, TRAN_VALUE, 0.0f));

item_g_model_matrix = glm::mat4(1.0f);
item_g_model_matrix = glm::translate(item_g_model_matrix, glm::vec3(TRAN_VALUE, TRAN_VALUE, 0.0f));
```

<sub>**Code Block 1**: Translating both items in the world space.</sub>

What happens when our character picks up the item, though? The item ceases to follow its own "space", and adopts the character's:

```
                   +
                  /
                 /
            +---+
            |   |
            +---+
—————————————————————————————————————
      Character + Item
```

<sub>**Figure 9**: Our character is now holding the item, and carrying it everywhere with them.</sub>

Our code now has to account for both the character's transformations **and** for the item's own transformations:

```c++
g_character_model_matrix = glm::mat4(1.0f);
g_character_model_matrix = glm::translate(g_character_model_matrix, glm::vec3(TRAN_VALUE, TRAN_VALUE, 0.0f));

item_g_model_matrix = glm::translate(character_model, glm::vec3(TRAN_VALUE, TRAN_VALUE, 0.0f));
item_g_model_matrix = glm::rotate(g_character_model_matrix, ANGLE, glm::vec3(0.0f, 0.0f, 1.0f));
```

<sub>**Code Block 2**: The character is probably still translated with respect to the world space, but our item now also has to be translated with respect to our character; in a sense, the character's space becomes the item's world space.</sub>

---

Now, it goes without saying that games require movement to actually be games. What we have been doing so far is something akin to the following:

```c++
glm::mat4 g_model_matrix;

void initialise() 
{
    g_model_matrix = glm::mat4(1.0f);
}

void update()
{
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(0.1f, 0.0f, 0.0f));
}
```

<sub>**Code Block 3**: Your basic game look animation.</sub>

This is how games have been programmed and thought of since the dawn of the industry—but it is now, by and large, antiquated and can cause weird things to happen.

Take as an example something that happened to several people in the class: their triangle was either spinning slower than mine, or oftentimes much faster than mine. This is not uncommon—after all, mine is a laptop from 2022 that is in no way set up to provide a crazy framerate. This is a problem, though, because when we make games, we want them all to run at the same "speed," regardless of the machine that it is running on. For this reason, we will change two things in our code. 

The first is, instead of keeping track of the matrix's transformations per frame, we keep track of the arguments values being passed into them:

```c++
float g_model_x;
float g_model_y;

void initialise() 
{
    g_model_x = 0.1f;
    g_model_y = 0.0f;
}

void update()
{
    glm::mat4 g_model_matrix = glm::mat4(1.0f);
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_model_x, g_model_y, 0.0f));
}
```

<sub>**Code Block 4**: A better way to handle position changes.</sub>

The second is, instead of relying on our computer's speed to update our frame, we need to use a method that will keep track of time the same way on every machine.

<br>

<a id="delta"></a>

### Part 3: _Timing, FPS, and Delta Time_

Faster hardware updates more times than its slower counterpart. This means that, unfortunately, if your computer is running at 60 frames-per-second (fps) and mine at 30 fps, the game _will_ run twice as fast on your machine than in mine.

```c++
float g_x_player = 0.0f;

void update()
{
    /**
    * This line of code will run more times on a faster machine.
    * At 60fps, it will run 60 times per second, for example.
    */
    g_x_player += 1.0f;
}

void render()
{
    glm::mat4 g_model_matrix = glm::mat4(1.0f);
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_x_player, 0.0f, 0.0f));
}
```

---

The way we standardise all of our players' play speed is by using something we call **delta time**. 

In game development, delta time (often abbreviated as `dt`) refers to the time elapsed between the current frame and the previous frame. Using delta time is crucial for ensuring that your game's behavior is consistent across different hardware and frame rates.

#### Why Use Delta Time?

1. **Consistency**: Without delta time, the game would run at different speeds on different machines. For instance, a game might run faster on a high-end computer and slower on a low-end computer.
2. **Frame Rate Independence**: Delta time allows your game to run consistently regardless of the frame rate. This means that whether your game is running at 30 FPS (frames per second) or 60 FPS, the in-game events will occur at the same speed.

#### Basic Concept

The core idea is to multiply any time-dependent variable (like velocity or movement) by `delta time`. This ensures that the game logic is consistent, no matter how much time has passed between frames.

Let's break it down with a simple example: moving an object in a 2D space.

#### Without Delta Time

If you move an object by 5 units per frame, the movement is tied directly to the frame rate. Here's an ASCII diagram illustrating this:

```
Frame Rate: 30 FPS

Frame 1: Object at position (0, 0)
Frame 2: Object at position (5, 0)
Frame 3: Object at position (10, 0)
...

Frame Rate: 60 FPS

Frame 1: Object at position (0, 0)
Frame 2: Object at position (5, 0)
Frame 3: Object at position (10, 0)
Frame 4: Object at position (15, 0)
Frame 5: Object at position (20, 0)
...
```

In this case, the object moves faster on a higher frame rate, making the game inconsistent across different hardware.

#### With Delta Time

If you move an object by 5 units per second, and use delta time to scale the movement, the object's movement will be consistent:

```
Frame Rate: 30 FPS (dt = 1/30)

Frame 1: Object at position (0, 0)
Frame 2: Object at position (0.1667, 0)   // 5 * (1/30)
Frame 3: Object at position (0.3333, 0)   // 5 * (2/30)
...

Frame Rate: 60 FPS (dt = 1/60)

Frame 1: Object at position (0, 0)
Frame 2: Object at position (0.0833, 0)   // 5 * (1/60)
Frame 3: Object at position (0.1667, 0)   // 5 * (2/60)
Frame 4: Object at position (0.25, 0)     // 5 * (3/60)
...
```

In this case, the object moves at a consistent speed regardless of the frame rate.

Using delta time ensures that your game logic is consistent and independent of the frame rate. It allows your game to run smoothly across different hardware, providing a better experience for all players.

Delta time can be visualized as the small time intervals between frames that you use to scale your updates, making sure the game behaves uniformly regardless of how fast the frames are being processed.

This calculation and value will look different, of course, depending on the machine. For example:

- **60fps**: Sixty frames per second means that your computer is changing frames every 16.66ms, making the delta time _0.0166_.
- **30fps**: Sixty frames per second means that your computer is changing frames every 33.33ms, making the delta time _0.0333_.

Our code above would thus change to the following if we were running at 30fps:

```c++
float g_x_player = 0.0f;
float g_z_rotate = 0.04f;
float g_delta_time = 0.0333f;  // But how do we calculate this? We're about to find out

void update()
{
    // This also works with rotation!
    g_x_player += 1.0 * g_delta_time;
    g_z_rotate += 0.001 * g_delta_time;
}

void render()
{
    glm::mat4 g_model_matrix = glm::mat4(1.0f);
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_x_player, 0.0f, 0.0f));
    g_model_matrix = glm::rotate(g_model_matrix, g_z_rotate, glm::vec3(1.0f, 0.0f, 0.0f));
}
```

<sub>**Code Block 5**: Using delta time to accommodate for a player's framerate.</sub>

---

Let's apply this delta time to a simplified version of our triangle program. Let's say that we just wanted to move the triangle 0.01 units to the right every frame:

```c++
/* Some code... */

float g_triangle_x = 0.0f;

/* More code... */

void update()
{
    g_triangle_x += 0.01f;
    g_model_matrix = glm::mat4(1.0f);
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_triangle_x, 0.0f, 0.0f));
}

/* More code... */
```

<sub>**Code Block 6**: Leaving the frame rate to our computer.</sub>


This doesn't actually fix our problem, though, as the value of `g_triangle_x` simply increases by 0.01 as fast as the computer can refresh the frame.

The way we do this is by keeping track of our **`g_ticks`**. Ticks are basically the amount of time that has gone by since we initialised SDL—kind of like a stopwatch.

Start by creating a global variable to keep track of the `g_ticks` from the previous frame. Something like:

```c++
float g_previous_ticks = 0.0f;
```

The formula to then calculate the delta time every frame is as follows:

```c++
void update()
{
    float g_ticks = (float) SDL_GetTicks() / 1000.0f;  // get the current number of g_ticks
    float g_delta_time = g_ticks - g_previous_ticks;       // the delta time is the difference from the last frame
    g_previous_ticks = g_ticks;

    g_triangle_x += 1.0f * g_delta_time;                 // let's try a much higher value to test our changes
    g_model_matrix = glm::mat4(1.0f);
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_triangle_x, 0.0f, 0.0f));
}
```

<sub>**Code Block 7**: Leaving the frame rate to the laws of space-time.</sub>

![delta-trans](assets/delta_trans.gif)

<sub>**Figure 10**: This is starting to make a lot more sense.</sub>

---

By the way, what would happen if we applied this delta time concept to rotation now?

```c++
float g_triangle_x = 0.0f;
float g_triangle_rotate = 0.0f;
float g_previous_ticks = 0.0f;

/* Some code here... */

void update()
{
    float g_ticks = (float) SDL_GetTicks() / 1000.0f;  // get the current number of g_ticks
    float g_delta_time = g_ticks - g_previous_ticks;       // the delta time is the difference from the last frame
    g_previous_ticks = g_ticks;

    g_triangle_x += 1.0f * g_delta_time;
    g_triangle_rotate += 90.0 * g_delta_time;            // 90-degrees per second
    g_model_matrix = glm::mat4(1.0f);

    /* Translate -> Rotate */
    g_model_matrix = glm::translate(g_model_matrix, glm::vec3(g_triangle_x, 0.0f, 0.0f));
    g_model_matrix = glm::rotate(g_model_matrix, glm::radians(g_triangle_rotate), glm::vec3(0.0f, 0.0f, 1.0f));
}
```

![delta-spin](assets/delta-spin.gif)

<sub>**Figure 11**: Our long-desired behaviour. It is here!</sub>

Try switching the order of operations to `Rotate -> Translate` and see what it looks like!
