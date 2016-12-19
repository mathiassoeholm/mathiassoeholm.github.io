---
layout: post
title: Layer Masks in Depth
---

Layers play a central role in Unity. They are used by Cameras in order to only render parts of a scene, and by lights for illuminating specific objects. But here we’ll look at their use in physics, to control what gets hit by raycasts, sphere casts, overlap sphere, linecast and all that jazz.
It turns out that using layer masks can increase the performance of these things, especially if your scene contains a lot of colliders.

### Ones and zeroes

A layer mask is an int. In other words, it is a series of 32 bits, for example:

  0000 0000 0000 0000 0000 0101 0000 0000
  
Each bit can be seen as boolean, 0 for false and 1 for true. This essentially allows us to store 32 boolean values in a single int. Curiously enough, we can have up to 32 different layers in Unity.
  
![layers]({{ site.baseurl }}/images/layers.png)

So looking at the layer mask above and the Layers we’ve set up in Unity. Can you name the layers that are “enabled” in the layer mask?

There’s an easy way of constructing such a layer mask in Unity. For example we can write:

```cs
LayerMask.GetMask(“Enemy”);
```

GetMask returns an int, and if we try to write this value to the console, it just writes 512. If you’re familiar with binary numbers, this is probably what you would expect. The binary representation of 512 as a 32-bit integer looks as follows:

  0000 0000 0000 0000 0000 0000 0010 0000 0000
  
There’s a 1 at the 10’th position (index 9) from the right. If we look at our layers configuration above, we see that this matches the position of the Enemy layer.

### Bitwise operations

Using LayerMask.GetMask is not the only way to create a layer mask, and it might not be the most convenient way either.

Let’s say we didn’t know the name of the layer that we wanted to create a mask for. But we have a reference to a GameObject on this layer. We might write something like this:

```cs
var layerName = LayerMask.NameToLayer(enemyGameObject.layer);
var layerMask = LayerMask.GetMask(layerName);
```

It looks alright, but we can do better:

```cs
var layerMask = 1 << enemyGameObject.layer;
```

The layer property on GameObject returns the index of the layer, meaning a value in the range [0...31].

The << operator is one of the bit-wise operators in C#. This means that it operates on the level of individual bits. Specifically we’re using the left shift operator. It shifts all the bits of the left operand to the left by the number of positions specified by the right operand.

So if we look at the binary representation of 1:

  0000 0000 0000 0000 0000 0000 0000 0000 0001
  
We see that there’s a single bit “enabled” all the way to the right. If we shift the bits to the left by 9 positions, we’ll get the following:

  0000 0000 0000 0000 0000 0000 0010 0000 0000
  
You’ll notice that this is the layer mask for the enemy layer.

Remember that the smart thing about layer masks is that we can combine them. We might have two game objects, whose layers we would like to combine in a layer mask. One possible way is this:

```cs
var layerNameA = LayerMask.NameToLayer(gameObjectA.layer);
var layerNameB = LayerMask.NameToLayer(gameObjectB.layer);
var layerMask = LayerMask.GetMask(layerNameA, layerNameB);
```

A better way would be:

```cs
var layerMask = (1 << gameObjectA.layer) | (1 << gameObjectB.layer);
```

Okay, so what the heck is going on here? We are using a new operator (&#124;) which is the bitwise-or operator. It takes two bit patterns of equal length and performs the logical OR operation on each pair of corresponding bits. This operator is best explained with a few examples:

0|0 = 0
0|1 = 1
1|0 = 1
1|1 = 1
0010|1000=1010
0000 0100 1000 | 0100 0000 1000 = 0100 0100 1000

I hope you can see how we are able to combine two layer masks with this operator. You can think of it as combining the true values in the two layer masks, where 1 means true and 0 means false.

### A point about code readability

Before you get all fancy and use these bitwise operators everywhere, consider the case where we actually know the layer names:

```cs
var layerMask = LayerMask.GetMask(enemyLayerName, projectileLayerName);
```

This is a lot more readable than the above examples that uses bit shifting. However we can actually encapsulate this part in an extension method:

```cs
public static int GetLayerMask(this GameObject gameobject)
{
  return 1 << gameobject.layer;
}
```

If you are not familiar with extension methods, I suggest you check out Unity’s tutorial. They’re just a compiler trick for calling a static method, but making it look like a method call on the instance. Now we can write:

```cs
var layerMask = gameObjectA.GetLayerMask() | gameObjectB.GetLayerMask();
```

### Using layer masks

Say that you want to create a raycast that hits everything but the gameobject casting it, and the Projectile layer. Here’s how you can do it:

```cs
var mask = ~(gameObject.GetLayerMask() | LayerMask.GetMask(“Projectile”));
if (Physics.Raycast(transform.position, transform.forward, 100, mask))
{
    Debug.Log("Hit something");
}
```

Here we’re using the ~ operator, which flips all the bits. The result is an opposite mask, that enables all the layers that were disabled and vice versa.

After creating a layer mask, you might wish to check if a gameobjects layer is enabled in that mask. Here’s how you can perform that operation:

```cs
public static bool IsInLayerMask(int layerMask, int layer)
{
    return (layerMask & (1 << layer)) != 0;
}
```

Here we use the bitwise-and operator (&), which takes two bit patterns of equal length and performs the logical AND operation on each pair of corresponding bits. Here’s a few examples of using the bitwise-and operator:

  0|0 = 0
  0|1 = 0
  1|0 = 0
  1|1 = 1
  1010|1000=1000
  
Here’s a challenge: Can you implement the IsInLayerMask method by using the bitwise-or operator, instead of the bitwise-and?

### Resources
Official documentation on layers: https://docs.unity3d.com/Manual/Layers.html

Some best practices of physics, including some performance profiling on RayCasts: https://unity3d.com/learn/tutorials/topics/physics/physics-best-practices
