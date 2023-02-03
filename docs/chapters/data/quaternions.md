# Quaternions
We're going to start with quaternions, mainly because without understanding how to use them there is no point in moving forward with skinning or inverse kinematics. I'm going to be honest here, I have no clue how they work mathmatically and you really don't need to know either. What's important is learning how to use them along with how to trouble shoot some niche issues.

With a quick introduction, what are quaternions? They are one of several ways that we can represent 3D rotation. Pretty much any gaming engine will use these for rotational duties. So why?

1. No gimbal lock *( almost )*, meaning that your rotation will never get one of it's axis locked to another. This is an issue prone to **euler** rotation. I say *almost* because I've seen someone once demostrate that it's possible to cause gimbal lock but if I remember correctly the sequence was near impossible to happen.

2. Very easy to interpolate using linear blending, just remember to normalize !

3. Uses up very little data, just 4 floats worth. Euler only needs 3 but a 3x3 matrix needs 9. Adding 1 more float for the other two reasons is worth it.

## Building Intuition
I like the phrase "building intuition" when we talk about quaternions. It was discribed like that years ago when I first started and it feels insanely true. The main issue is that most people have a hard time learning the mathmatics behind how quaternions work. There is another method known as **Rotors** which is suppose to be easier to understand & functions the same as quaternions but no one uses them. Without support & a library for Rotors, we're stuck using quaternions for now.

Since it's hard to understand how they work, the only solution for most of us is to understand how to use them. We can just accept them as a black box of complicated math as long as they perform the functions we need them to do. The first thing to understand about quaternions is they function similarly to matrices. That means how you handle parent-child transforms, local & world space, the order you multiple and even the use of the inverse function is all about the same. So if you understand how to use a 4x4 matrix to handle transformations, then I guess you're done here... naw, I'm kidding. We got other goodies in this chapter that you can't easily do with matrices.

Lets go build up that intuition now, shall we.

### Components ( XYZW )
The first order of business is to know how many components is used to define a quaternion. It consists of 4 float values. The first 3 can be called XYZ, which holds the axis of rotation. If we're rotating on the X axis, the values will be something like [ 1, 0, 0 ], but not quite. The 4th float is called W which contains the angle of rotation on said axis, it's not in degrees or rads but the weirdly notion of sin( radian * 0.5 ). Does it really matter to know how W is computed? No, its just a point that the interal math of quaternions are complicated & you shouldn't go changing the components values directly of a quaternion without knowing what your doing or using a library. Life can be hard & scary if you try to manually change W expecting it to work, even with the sin stuff... its more complicated then that & you shouldn't attempt that suicide mission.

Depending on your library you will either interact with quaternions as Arrays or Objects. In some libraries you can interact in both ways if they follow an Object Orientated approach. When you deal with shaders, the vec4 will be your data type to hold quaternions but keep in mind shaders don't have any built in operations for quaternions. You'll need to include GLSL functions into your shaders to do those transformations.
```js
const aryQuat = [0,0,0,1];              // Array Form
const objQuat = { x:0, y:0, z:0, w:1 }; // Object Form
```

Some might ask, Why is W after Z? Well, its for code simplicity. Quaternions have XYZ components, so does a Vector3. If we're dealing with array form of things, it'll be better if the indices are the same. Its easier to just add an extra component at the end and call it something, why not W? I have seen notations in the past where people write quaternions as [ W, X, Y, Z ], not sure if there are any libraries out there that uses that order but I bet there is at least 1.

Since you'd want to avoid directly accessing or modifing the components it does not matter the order unless your trying to pass a quaternion to a shader. At that point you should really know the order your library is using and get the right shader functions that support that order or just modify the order before sending it to a shader.

### The Identity
You'll see this word from time to time when reading this ebook. The identity usually means the default value that doesn't do anything. Like for scaling, the identity is 1 because it doesn't change anything when you use it. For quaternions the identity is [ 0, 0, 0, 1 ] with the axis part all zeroed out with a 1 in the W component. When you apply this rotation, it causes no change.

When you use a library like gl-matrix, I'll often just initialze a quaternion with an array with the identity instead of using the quat.create() method. It's one less function call.
```js
const q = [0,0,0,1];
```

### Local & World Space
Spaces are always a hard thing to describe & harder for beginners to wrap their heads around. This is only a thing when you're dealing with a hierarchy of transformations. If we have a single quaternion to represent the rotation of an object it technically lives in both spaces or no space at all. 

If you like anime, I would visualize each "local space" rotation as a power up, you can keep stacking up all your LS with world space representing your current power level & physical form. Mind you, this isn't even my final form :)

[ PICTURE OF FREEZA AND ALL HIS TRANSFORMATIONS ]

For non-anime junkies, local space can be seen as robot parts with world space representing the whole android. Like the phase "The sum of all its parts".

Math-wise, how about 3 stick that are 2 inches tall. If all your sticks are bundled together and pointing up, you can say each stick has the same starting position and ending position in the world. Now stack the sticks one ontop of the other, from end to end. The chain of sticks still have the same starting position from the ground but not the same ending position. The first stick has the world space end position of 2 inches, the second stick is 4 inches and the final stick is 6 inches. This is the general idea of local space & world space.

[ MAKE AN IMAGE OF STICKS ]


## Operations

### Multiply
Lets begin with how we can add up quaternions, and yes, we multiply them to basicily add then up in many cases. The main case is parent-child relationships. To get world space rotation you first need to know the world space rotation of the parent object, then you multiply it by the local space of your child.

You can visualize it like a rubics cube. Each local space rotation is a turn on solving the puzzle. The world space is the current state of the cube at each turn.
```js
const q0_ls = new Quat();    // Localspace rotations, each 1 relative to world origin
const q1_ls = new Quat();
const q2_ls = new Quat();
const q0_ws = q0_ls;         // Yes, they are the same when its the first/root rotation
const q1_ws = q0_ws * q1_ls; // Worldspace Rotation of Q1
const q2_ws = q1_ws * q2_ls; // Worldspace Rotation of Q2
```

### Inverse
You can use inverse of a quaternion to get the reverse version of it. Another use case is getting a difference between two quaternions, which is very helpful to perform world space to local space conversion. You can view this like a **'subtraction'** operation in a way. Since the inverse is a reverse rotation, its like your rewinding the another quaternion to a state where the rotation's origin is the inverse quaternion.

```js
// Theory in an algbebra way
const c  = a + b;
const b  = c - a;

// How its done with quaternions
const qc = qa * qb;
const qb = invert( qa ) * qc; // Order is important, rewind QC before QA existed
```

If we continue with the multiply example, you can see how to use invert to convert from world space back to local space.

```js
const q0_ls = new Quat();    // Localspace rotations, each 1 relative to world origin
const q1_ls = new Quat();
const q2_ls = new Quat();
const q0_ws = q0_ls;         // Yes, they are the same when its the first/root rotation
const q1_ws = q0_ws * q1_ls; // Worldspace Rotation of Q1
const q2_ws = q1_ws * q2_ls; // Worldspace Rotation of Q2

// ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
// Inverse to get local space rotation of Q2
const q1_wsInv = invert( q1_ws );   // Invert the parent WS rotation
q2_ls = q1_wsInv * q2_ws            // recreate the original LS rotation
```

Finally you can use inverse to get the difference to translate rotations. When you handle retargeting animations from one skeleton to another, you would create a rotation difference for each bone shared between both. When the bone changes you can use this difference to translate that rotation to one that works on the other skeleton.

```js
const srcHip        = source_tpose_hip_rotation;    // Default rotation from TPose
const targetHip     = target_tpose_hip_rotation;
const hipTranslater = invert( srcHip ) * targetHip; // difference from source to target

// Change the Hip Rotation
srcHip.rotateX( 90 );

// Check for possible hemisphere error ( explained later in Tips & Tricks )
tmpSrc = ( dot( srcHip, hipTranslater ) < 0 )? negate( srcHip ) : srcHip;

// Translate the change in the source hip to work correctly on the target hip
// In this case, the difference is applied as the right side term of the operation
targetHip = tmpSrc * hipTranslater;
```

Lastly, when you want to apply world space rotation on another world space rotation you would put the transforming quaternion as the left term of the operation.
```js
const turnRight = quat.setAxisAngle( [0,1,0], 90 * toRad );
let   rotHead   = some_forward_looking_rotation;

// Head should now be looking right instead of forward.
rotHead         = turnRight * rotHead;
```

## Tips & Tricks

### Normalizing
This is uberly important. When you start multiplying many quaternions together, things start to drift a lil. You don't need to normalize a quaternion after every operation, doing it before appling it to a model is more then enough. When you see weird artifacts in skinning, I would try normalizing the final quaternion thats being applied to the vertice first. This quite often fixes things.

```js
function normalize( q: quat ): quat{
    const mag = Math.sqrt( q.x*q.x + q.y*q.y + q.z*q.z + q.w*q.w );
    if( mag === 0 ) return q;
    
    // mul is faster then div, invert for optimization
    const invMag = 1 / mag;  
    q.x *= invMag;
    q.y *= invMag;
    q.z *= invMag;
    q.w *= invMag;

    return q;
}
```

### Mirroring
Mirroring quaternion is something you rarely see in a library, but it's an operation you can do pretty easily. The main idea is that whatever axis you want to mirror, you negate the other axis components. 

```js
function mirrorX( q: quat ): quat{
    q.y = -q.y;
    q.z = -q.z;
    return q;
}
```

[ ADD PICTURE SHOWING TWO CUBES ]

### Dot & Negate
This is a life saver. There are times you see some bad rotation behavior or artifacts when applying your rotation. There is an issue that can easily happen when you multiply quaternions together & when you're dealing with animations or skinning it can cause some sort of ***glitch in the matrix***. Gabor describes the problem as one rotation is taking the longer path around toward the other instead of the shorter path. I like to visualize it as the rotation axis of both quaternions are on opposite hemispheres of a sphere. Either way you look at it, this is not something can be fixed with normalizing the quaternion.

[ MAKE IMAGE OF A SPHERE WITH TWO VECTORS ]

The first step is to do a dot product on the two quaternions. If you get back a negative value then all you need to do is pick a quaternion to negate. Negating a quaternion flips the axis & rotation but gives you the same result minus the error when you multiply it with the another. So this causes the two axes to be on the same side of the sphere, causeing the path of rotation to be the short one.

```js
const a = new Quat();
const b = new Quat();

if( dot( a, b ) < 0 ) negate( a );

const c = mul( a, b );
```

Since this is such an important operation, I've always have a function available to perform this check & operation for me. Trust me, this was a show stopper in the beginning of my skinning development years ago till Gabor told me about dot check & negating as a possible fix. Which it did, vegeta's butt vertices stopped exploding when I would rotate his left leg.

```js
/** Checks if on opposite hemisphere, if so, negate the first quat */
static dotNegate( out: quat, q: quat, by:quat ): quat{ 
    if( quat.dot( q, by ) < 0 ) this.negate( out, q );
    return out;
}

static negate( out: quat, q: quat ): quat{
    out[ 0 ] = -q[ 0 ];
    out[ 1 ] = -q[ 1 ];
    out[ 2 ] = -q[ 2 ];
    out[ 3 ] = -q[ 3 ];
    return out;
}
```

?> Which quaternion to negate doesn't matter. Ideally the one that isn't a reference to an object as you may not want to change the value.

?> Beyond skinning, I've also fixes issues related to using springs on quaternions when I switch the target value. 

### Orthogonal Axes
I use this often when I need a very specific rotation. If you don't know how 3x3 rotation matrices works, then here's a quick primer. You can define rotation with a matrix by placing vectors that are orthogonal in the right column/row. Orthogonal just means each vector is 90 Degrees from eachother. For quaternions, we need these vectors to be normalized.
```js
Row Major or Column Major, Identity is the same... No rotation.
           X  Y  Z 
RIGHT   X  1, 0, 0 
UP      Y  0, 1, 0
FORWARD Z  0, 0, 1
```

If lets say I want my rotation to point up, I can set Forward( Z ) as [ 0, 1, 0 ] which is the up unit vector. From there I need an UP( Y ) so lets pick back as [ 0, 0, -1 ] which is the inverse forward unit vector. Now we need a RIGHT( X ) which we can keep the same [ 1, 0, 0 ]. As long as each vector is normalized you can perform a mat3x3 to quaternion conversion. When you apply this rotation to an axis helper mesh of some kind you'll see that forward is pointing up and up is point back.

A useful example of this is creating a **Look** rotation.

```js
static look( dir: vec3, up: vec3 = [0,1,0] ): quat {
    const zAxis = vec3.copy( [0,0,0], dir );            // Forward, keep dir const
    const xAxis = vec3.cross( [0,0,0], up, zAxis );     // Right, its orthogonal to Z
    const yAxis = vec3.cross( [0,0,0], zAxis, xAxis );  // Realign Up, orthogonal to XZ

    // Make sure to turn them into unit vectors
    vec3.normalize( xAxis, xAxis );
    vec3.normalize( yAxis, yAxis );
    vec3.normalize( zAxis, zAxis );

    const m3 = [ ...xAxis, ...yAxis, ...zAxis ];
    const q  = quat.fromMat3( m3 );

    return quat.normalize( q );
}
```

### Swing & Twist
This is a set of operations that handles many of the IK movements in my animation library. The premise is to create 2 seperate rotations that you can apply to a bone. First you create a swing rotation that moves from one unit vector to another. Once you apply the swing, you compute the axis you want to twist on to create of the twist quaternion using a axisAngle operation. You can think of it like swinging your arm, then twisting your hand to point up.

```js
// Forward is [0,0,1], Up is [0,1,0];
const q = [0,0,0,1];

// Create a rotation that swings FORWARD to point RIGHT
const swing = quat.rotationTo( [0,0,1], [1,0,0] );

// Apply swing, Forward is [1,0,0], Up is [0,1,0]
q = swing * q;

// Find the current Forward Axis to use at the twisting axis
// The axis should be [1,0,0];
const twistAxis = transformQuat( [0,0,1], q );
const twist     = quat.setAxisAngle( twistAxis, 90 * toRad );

// Forward is pointing Right [1,0,0]
// Up is pointing back [0,0,-1]
q = twist * q;
```
[ MAYBE ADD ANIMATED GIF SHOWING THIS OPERATION ]

The only issue is the swing rotation can have a lil bit of twist already attached to it after being applied. We can dive deeper by constructing a swing rotation that kind of resets the twisting or more like it forces the UP direction to be a specific direction. It's something new I've been experimenting with & would probably be part of future chapters.

### Inverse Directions
This is one of the secret ingredients to the way I perform inverse kinematics. When you deal with a skeleton in a T-Pose, the bones arent axis aligned & the twist rotation can be all over the place. Just like penicillin, I accidently came up with this... it was a math error on my part with wonderful results.

The main idea was to get bones axis aligned without messing up the T-Pose. If you take the inverse of the worldspace rotation of the bone then multiply it by what axes you care about it would generate directions that will become axis aligned when applying the world space rotation. When the bone moves, these inversed directions will continue to point at the proper direction of where the bone points.

```js
const fwd    = [0,0,1];                  // The direction we care about
const q      = fromEuler( 27, 50, 83 );  // funky rot with forward not being forward
const qi     = invert( q );
const invFwd = transformQuat( fwd, qi ); // get inverse direction, not pointing fwd now 

q   = rotateY( q, 90 );             // Rotate Quat by 90 degrees on Y
fwd = transformQuat( invFwd, q );   // Transform Inverse Forward by new rotation

// Forward is now pointing Right
fwd === [1,0,0]  // TADA, Magic!
```

### Order of operations
Since i'm not a master of quaternions, when i'm doing something new I may find that things aren't rotating how I expect them to. Sometimes a quick fix is just reversing the order. So q0 * q1 becomes q1 * q0. When in doubt just reverse the order & see if that fixes the problem. I can't count how many times this resolved my problems over the years, same goes with matrices :)

### Individual Axis Rotations
When rotating things with quaternions in an euler like manner, you need to rotate things in a specific order for things to look right. Not sure if this changes based on the library, but for gl-matrix this is how I apply rotations per axis when I prototype things.

```js
const q = [0,0,0,1];                // Identity - No Rotation
quat.rotateY( q, q, 37 * toRads );  // First rotate by Y - Turn
quat.rotateX( q, q, 77 * toRads );  // Then by X         - Tilt
quat.rotateZ( q, q, 90 * toRads );  // Finally with Z    - Twist
```

### Compute World space from Leaf to Root
If you have a tree of bones or whatever, you will often want to quickly compute the world space rotation. Normally you would compute such a thing from parent to child down the tree. There's a trick to do it from
child all the way to the root. If we start at some child rotation, we can then work backwards by using the parent's local rotation multiplied by the world space build up. It'll probably make more sense with code.

```js
function worldSpaceRotation( leafBone: Bone ){
    // We have no parent, local is world.
    if( leafBone.parent === null ) return leafBone.localRot;

    // Lets start with the bone's local rotation
    const wsQuat = leafBone.localRot.clone();
    let   bone   = leadBone;

    while( bone.parent !== null ){
        // Move to the parent bone
        bone   = bone.parent;

        // We still keep the parent * child multiply order but the data is in reverse
        // Normalally it's parent.world * child.local but in this 
        // use case its parent.local * child.world.
        wsQuat = bone.localRot * wsQuat;
    }

    return wsQuat;
}
```

## References
- https://gabormakesgames.com/blog_quaternions.html
- https://marctenbosch.com/quaternions/