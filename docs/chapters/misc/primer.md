# A primer on Inverse Kinematics

The concept of modern day inverse kinematrics was originally introduced by D.E. Whitney in a paper he  wrote back in 1973 titled, "Inverse kinematic problem for a robot manipulator". His work was build around using the jacobian matrix as an interive approach to compute the proper rotations for each joint that would allow the arm to reach a specfic a position. Whitney's work wouldn't be possible without Richard Paul who was the first to tackle this robotics problem and who is responsible for the ground work of analytical & numerical methods still used today.

As time progressed, IK which was something originally created for robotics became a popular solution for early animators in the film and video game industry. For general animation IK provides a way for tool developers to create ways to make it quick & easy to pose a characters in a scene. Instead of manually adjusting every bone, now animators can easy just a single point around and allow the ik solvers to figure out all the rotational axes & angles for the user.

When it came to gaming, it provided ways to procedurally animate characters to make them more believable in the virtual worlds they inhabit. For example being able to reach out to a walk when near it or properly placing feet when walking up stairs.

### What are analytical methods?
<img align="right" width="350px" src="./chapters/misc/img/tri_arm_ik.png" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

In the beginning of robotics, controlling joints was a very manual process akin to what animators would call forward kinematrics ( FK ). Any animator can contest that FK can be a tedius & harder way to pose a character for animations and robotic enginners of the time thought so too. This is where Richard Paul came in and started using the concept of a target position while using geometric or algebraic solutions to control the movement of simple robotic arms.

For example, the human arm and leg can have a analytical ik solver by simply reducing the problem down to a triangle. Knowing the length of the upper bone ( humerius ) & the lower bone ( ulna ) satisfies 2 of the 3 lengths we need. If we take the distance of the shoulder to our target position that will work as our third. Knowing the lengths of 3 sides of a triangle is whats required to solve all its angles by using the law of cosines. 


### What are iterative methods?

<img align="right" width="350px" src="./chapters/misc/img/ccd_algo.gif" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

Iterative methods came about when advanced movements and complex robotic arm configuration was needed. As with analytics needing a perfect mathematical formula, we can instead take small steps & adjustments toward the target position for the best approximate set of rotations for each joint. This became essential when dealing with enviroments like reaching around an obstacles or having a moving target. 

The image on the side is demonstrating how the **Cyclic Coordinate Descent ( CCD )** solver operates by adjusting each joint to match the end effector's direction torwards the target. Beyond CCD, other popular solvers in this category are **Jacobian Method ( Differential IK )** and the very popular **Forwards & Back IK ( FABRIK )**.


## Animation Retargeting

<img align="right" width="350px" src="./chapters/misc/img/at_pose.png" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

Retargeting is a process where a character pose is copied to another character. If the characters are created from the same skeleton, then a simple copy is all you need. In general that isn't often the case. Characters come in various poses, the main two is called APose and TPose. Each pose type has it's stength & weaknesses. 

BUT For retargeting we need a TPose but its ok if your character is in an APose. A solution to remedy this is by using whatever modeling tool you prefer and create a single frame animation where you set your character in a TPose & export it out with your APose character. This will work just fine for retargetting while you can keep your APose as the main bind pose, eliminating any need of reskinning your character into a TPose.

Why a TPose? Well, its world axis aligned which will make the math simpler for the process. The issue is that the APose comes in various angles for the limbs as there isn't a "Standard" shape. Sometimes the arms are closer to the hips, sometimes the legs are more like a TPose or spread out quite alot. A 20 degreee change from one APose will be different for another because of these variantions. By using a TPose, a 20 degree rotation on the Z axis will give you the same or an approximate rotation on your target bone. This works because you know both skeletons have the same basic shape & general alignment.

There is still one problem, even though you have two TPoses for retargeting, bones can have any random orientation be it proper alignment from parent to child or just zero rotation on all bones. To make retargetting work we need to setup a form of a translator from your source bone to your target bone. Since we have the same "shape" of the skeleton, this translator is to handle the differences in orientation.

#### Setup link between bones
For every bone I would compute & cache a quaternions that will be reused for each bone when doing animation retargetting.

- Source Bone's parent worldspace rotation from TPose<br>
  **Source_WSParent = srcBone.parent.world.rot**

- Source Bone's worldspace rotation from TPose, used for DOT Check<br>
  **Source_WS = srcBone.world.rot**

- Target Bone's parent worldspace inverted rotation, for local space transformation<br>
  **Target_WSParentInv = invert( tarBone.parent.world.rot )**

- Convert source worldspace to target worldspace<br>
  **Source2Target = invert( Source_WS ) \* tarBone.world.rot**

#### Retargeting Transformation Steps
1. Isolate changes of source bone after being posed or animated<br>
   **qDiff = Source_WSParent \* SourceAnimBone.local.rot**

2. Next do an orientation test to prevent artifacts when applying to vertices. If the two rotations are on opposite hemispheres of rotation, a quick fix is negating one of the quaternions. Negating is different from inverting, doing so doesn't change the 3D rotation because of the property of double cover of a 4D hypersphere. All this does is insures that rotation happens on the shorter way instead of the long way around. Without it, you can see oddities with operations like slerp, weighting or transforming vectors.<br>
   **qToTarget = ( Quat.dot( qDiff, Source_WS ) \> 0 )? -Source2Target : Source2Target**<br>
   **qDiff = qDiff \* qToTarget**

3. Rotation has been transfered to target bone in worldspace, now move it to localspace<br>
   **qFinal = Target_WSParentInv \* qDiff**

Thats pretty much all you need to do retargetting. This will work fine as long as the source and target skeletons are similar in shape. You can get away with a spline with a mismatch of bones but the recreating the animation will suffer. The only other tip for retargetting is to try to include the clavicle bone. There are many animations that uses that bone and excluding it can make the recreated animation look stiff or give you the feeling that something seems off about how the arms move.

## IK Retargeting

<img align="right" width="350px" src="./chapters/misc/img/skeleton.png" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

Ok, so what if your source & target skeletons are not similar at all? For example, what if your trying to apply a human motion to a werewolf that have 3 main joints in their legs instead of two? Or your source animation has 3 spine bones but your target has 8 for some reason? This is where inverse kinematrics comes in.

Like with any big problem its best to follow a divide & conquer approach. The general plan is to divvy up the body into various chains of bones, each with its own ik solver and target. 
1. **Limbs** - These are the easest to mange since they follow the same general mechanics by use of hinge joints between the start & end points. Depending on the amount of bones, various analytical solvers can be used.
2. **Head, hands/wrists, feet/ankle** - These follow similar motion to ball joints but with various limit constraints. An aim/look analytical solver seems to work well with theses.
3. **Spine** - This is a tricky one, as the solvers can be different based on the general application for example retargeting or posing. For retargeting I've found interplating quaternions based on directional axes to work well. When it comes to posing, a spline solver with parallel transport for the normal vectors seems to work well for this usecase. Jacobian method which is a slower solver but works well here because it can handle both rotational & positional targets. Digging in blender's source code I've see that it uses this solver by default for many of its IK operations.
4. **Pelvis** - This is the oddball. For posing, FK is all you need since IK would be overkill. For retargeting this bone actually needs two solvers each one respectively handling rotational & positional problems.

#### Directional IK Targets

<div style="width:350px; float:right; display:flex; flex-direction:column; gap:10px;">
<img width="350px" src="./chapters/misc/img/ik_dir.gif" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

<img width="350px" src="./chapters/misc/img/ik_gdc_chains.gif" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">
</div>

Ubisoft had a GDC talk back in 2018 where they first demostrated how they use inverse kinematrics to retarget & modify animations from a humanoid to various other characters & animals. Normally when dealing with IK we use a target position that we wish an arm or leg to reach towards. This works great for posing or making a character more interactive in the scene, but not exactly useful for retargeting. This is where direction data comes into place. 

Lets take the arm as an example and break it down into various bits of information that we can use to retarget its current pose. First we know where the shoulder & hand exists in world space, from that we can derive a unit vector direction and the distance between the two points. Distance by itself isn't very usuable as each character will have various lengths but we know each arm will have a maximum reach. If we take the sum length of the arm when its fully extended, we can use it to normalize the current reach distance to create a scalar value. The last piece we ned is some pole or bend direction which can be computed in various ways. So two unit vectors & a scalar value is all we need to recreate the general shape of any arm or leg.

```js
// Compute IK Data from source
toHand  = srcHand_WSPos - srcShoulder_WSPos 
dist    = Vec3.length( toHand )
ikDir   = Vec3.normalize( toHand )
ikScale = clamp( 0, 1, dist / srcArmChain.maxLength )

// Apply to Target
ikPos = targetShoulder_WSPos + ikDir * ( ikScale  * targetArmChain.maxLength )
ikSolver( targetArmChain, ikPos, finalPose )
```

A good amount of IK retargeting, at least the things I've created can be done pretty much with just orthogonal unit vectors. Its easier to understand and visualize as a stand in for rotation. IK Target Direction can be replaced by the **forward / z vector** with the Pole Direction replaced by the **up / y vector**, we now have a general sense of rotation. Forward can be used to perform **swing** rotations with up controlling the **twisting** rotation. When we need to create a rotation, all we'll need to do is create one more vector which is our **x_vector = cross( forward, up )**. As long as the 3 vectors are normalize & orthogonal to eachother, then you have a 3x3 rotation matrix ready to go. Converting the vectors aka 3x3 matrix into a quaternion is usually available in math libraries.


#### Inverse Axes
Just like regular retargeting, we still have the problem of each bone having various orienations. For IK retargeting I stumbled into a solution that I call inverse axes. The general idea is that we target each world space rotation of each bone with an axis aligned directions that are then transformed by the inverse rotation. Sounds complicated buts its us stating that whatever orientation this is lets declare it as zero rotation.

```js
// Setup our Inverse Direction
q          = random_quaternion()
invForward = [0,0,1] * Quat.invert( q )

// Modify our quaternion by rotating it by 90deg on the X axis
// Then transform our inverse direction.
// Now forward's value is [0,1,0], it's point up while not 
// knowing what is the true orientation of the quaternion
q          = Quat.axisAngle( [1,0,0], 90 ) * q
forward    = invFoward * q;
assert( forward === [0,1,0] )
```

Take this idea and say the left thigh bone's forward direction is **-Y [0,-1,0]** and its up direction is **+Z [0,0,1]**. With a cross product we get our right direction as **+X [1,0,0]** for free. We can do this for both the source & target skeletons that are in a **TPose**. Now we have a 1:1 directional axes on both skeletons & we dont even know nor care of the initial orientation. Now all our work can be done with vectors that will define our swing & twist quaternions. A nice bonus is that we can always transform our inverted up direction by the current rotation to know the pole direction of the leg at any given time. 

So how will retargeting work using inverted directions? Our analytical ik solvers will use these inverted directions to make the target match the source. The math behind this is pretty straight forward by getting the rotation axis from the cross product of the two vectors along with the angle between them. Here's a function I found a long time ago and have been using to create my swing & twist quaternion rotations.
```js
/** Using unit vectors, Shortest swing rotation from Direction A to Direction B  */
quatSwing( a: ConstVec3, b: ConstVec3, out: QuatLike=[0,0,0,1] ){
    // http://physicsforgames.blogspot.com/2010/03/Quat-tricks.html
    const dot = Vec3.dot( a, b );

    if( dot < -0.999999 ){ // 180 opposites
        let tmp = Vec3.cross( Vec3.LEFT, a );
        if( tmp.len < 0.000001 ) tmp = Vec3.cross( Vec3.UP, a );
        Quat.fromAxisAngle( Vec3.normalize(tmp), Math.PI, out );
    }else if( dot > 0.999999 ){ // Same Direction
        Quat.identity( out );
    }else{
        Vec3.cross( a, b, out );
        out[ 3 ] = 1 + dot;
        Quat.normalize( out, out );
    }
    return out;
}
```

#### Handling Pelvis & Root Translations

Up to this points we've been dealing with directions & rotations when dealing with IK Retargeting. Translations can be tricky as it needs to be properly translated from one skeleton to another. It becomes very apparent when dealing with big size differences between our source & target skeletons. Plus the up & down position is extremely important to get current to make any walk, run or jump to look believable.

A general work around is to find a **scalar** between the two skeletons. Since most of the translation is based on stride distances as the character walks, we can use the distance from the pelvis to the ground as our normalizer. All thats left is to scale the delta movement of the source's pelvis position then add it to the target's TPose position.

```js
// Compute scale from world space y position of the pelvic bone from a TPose
// Equation : A * Scale = B, rewritten to Scale = B / A
const scalar = TargetTPose.pelvis.world.y / SourceTPose.pelvis.world.y

// Apply Animated Translation
const WSOffset = SourceAnimPose.pelvis.world.pos - SourceTPose.pelvis.world.pos;
TargetAnimPose.pelvis.world.pos = ( WSOffset * scalar ) + TargetTPose.pelvis.world.pos;
```

## Spring Chains

<img align="right" width="350px" src="./chapters/misc/img/spring_chain.gif" style="border:4px solid #202020; margin:0px 0px 0px 8px; border-radius:10px;">

Now that you can retarget a human walk cycle onto a TRex with the right IK solvers, what about it's tail? A popular solution to add automatic procedural animations is by using a spring physics. The general idea is that our TRex's hips will be animated in some fashion then a spring system will cascade that motion done all the bones of the tail, like tipping dominos.

When it comes to springs, there are different types of algorithms that can be use. Some of my favorites are **hookes law**, **dampers** and **implicit euler**. Each one has its uses but I often stick to **implicit euler** as its the easiest to dial in the sort of motion I'm looking for. As a side note, its also the main spring algorithm for the Box2D Physics Engine.

#### How does it work
Just like with IK, you setup a chain of bones. Each bone will have a target position, in my system I use the inverse axes since I can control the direction I want to work with without needing to worry about the bone's initial rotation. Beyond this target position, you'll need all the property for the spring algorthim of choice. Some properties you may encounter  are mass, tension, dampening, oscillation per second, half life, etc.

Next you need to keep track of the current spring position & velocity along with the target or resting position Those are the bare bones but some agorithms may need to track accelleration too. Beyond that, here's some pseudo code of the spring operation on a single bone.

```js
// Compute the resting rotation of the bone in world space
rest_rot    = bone.parent.world.rot & tpose.bone.local.rot
target_pos  = rest_rot * inv_direction

// Run your spring
applySpring( out spring_vel, out spring_pos, target_pos, deltaTime );

// Create a swing rotation from target vector toward spring vector
// Then apply this rotation to our computed resting rotation
swing  = quatSwing( Vec3.normalize( target_pos ), Vec3.normalize( spring_pos ) )
ws_rot = swing * rest_rot;

// Move from world space to local space
local_rot = Quat.invert( bone.parent.world.rot ) * ws_rot;
```

When doing further research on the topic its good to know that this feature comes in many names,  the main ones are **Jiggly Bones**, **Wiggly Bones**, and **Spring Bones**.

## References
- Ubisoft IK Rigs GDC Talk<br>
https://www.youtube.com/watch?v=KLjTU0yKS00

- Implicit Euler Springs<br>
http://allenchou.net/2015/04/game-math-precise-control-over-numeric-springing/<br>
http://allenchou.net/2015/04/game-math-more-on-numeric-springing/