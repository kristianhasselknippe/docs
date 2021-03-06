# draw

The draw keyword is used to draw graphic primitives to the current framebuffer based on parameters derived from the draw path.

For a practical guide on using the draw keyword refer to the development guide.

## Syntax

```
draw [ <block1> [, <block2> ...]] [ { <inline block> } ] ; 
```

The draw keyword indicates the start of a draw statement. A draw statement consists of the draw keyword followed by zero or more block references or lambda blocks, and terminated by an optional a single inline block and a semicolon.

The referenced blocks will be applied, in order. This means that:

```
draw Foo, Bar;
```

Is equivalent too:

```
draw { apply Foo; apply Bar; };
```

See the documentation on the apply keyword for more information about what this means.

When block references are object references, they can be prefixed with the keyword virtual to imply that the run-time type of the object should be applied instead of the compile-time type of the reference variable. This is called virtual apply.

### Remarks

When the compiler encounters a draw statement, it will generate zero or more draw calls for the underlying graphics API. The draw call is expected to be executed once or more during the life time of the containing object. To accommodate the draw calls the compiler may insert additional private (and invisible) fields and methods in the class containing the draw statement. The inserted fields will hold data and objects that are useful across multiple executions of the generated draw calls, such as vertex buffers, shaders and shader parameters.

The compiler will to the best of its ability and in light of best practices of the target platform and underlying graphics API try to reuse data, shaders, shader parameters, buffers, textures and other resources between all generated draw calls in a project. The compiler may also try to merge (batch) multiple draw calls generated by a draw statement into a single or fewer draw calls to the underlying graphics API.

The draw calls may be initialized partly in a static context, and partly when the containing class is instantiated. For this reason the draw keyword can only be used inside non-static methods of classes.

The draw calls are not guaranteed to be initialized until the end of the containing class' constructor. Hence it is not allowed to use draw statements in methods of a class while its constructor is being executed. Doing so will throw an exception at runtime. Note that this only applies to draw statements contained wihtin methods of the object being constructed. A constructor may very well invoke methods on other objects that involve draw statements.

The lifetime of the data for the draw operation is assumed to be bound to the lifetime of the class containing the draw statement, and will thus be garbage collected along with the containing object.

## Behavior

The draw path is an ordered list of blocks, given by:

1. The reverse order of the blocks in the draw statement, followed by
2. all blocks found when seraching upwards from the beginning of the current method to the start of the class declaration, visiting all nodes in that path, if any, followed by
3. all blocks found when recurisvely visiting all base classes, searching upwards from the bottom of the class declaration, visiting all nodes found in that path, if any.

When the compiler encounters a draw statement, the following happens:

* The compiler constructs the statement's draw path, as described above.
* The compiler searches the draw path for all drawables.
* If, and only if, no drawables are found, the compiler inserts an implicit drawable.
* For each drawable, a draw call is generated by resolving all terminal properties from meta properties in the draw path, and finally emitting appropriate API calls to the underlying graphics API.

Terminal properties are parameters to the graphics pipeline, such as render states, data buffers and shaders. There is a large set of terminal properties that can be modified, but only two are stricly required:

* ClipPosition - an expression describing the output from the vertex shader as a float4 vertex position in clip space.
* PixelColor - an expression describing the output from the pixel shader as a float4 color in the RGBA color space.

## Block references

Block references can be:

* Any qualified name to a declared block elsewhere in the code.
* Any local variable or field accessible in the scope.

A call to a constructor of a block factory class, such as Uno.Content.Models.ModelBlockFactory. When calling block factory class constructors, the 'BlockFactory' suffix must be ommitted, e.g. we can simply call 'Model' instead of 'ModelBlockFactory'.

### Examples:

```csharp
// Using a qualified name to locate an external block
draw SomeNamespace.Sphere;

// Using a local variable object reference as a block
var foo = new Foo();
draw foo;

// Using a block factory class to draw a model
draw DefaultShading, Model("foo.DAE");
```

## Inline and lambda blocks

The draw statement supports a single inline block to be supplied as the only or last block in the statement, as well as any number of lambda blocks.

### Inline blocks

In the most basic case, a draw statement can be supplied only a single inline block that describes all neccessary terminal properties. The following example draws a red triangle using a single inline block.

```csharp
draw
{
    float2[] vertices: new float2[]
    {
        float2(-0.5f, -0.5f),
        float2( 0.5f, -0.5f),
        float2( 0.0f,  0.5f)
    };

    ClipPosition: float4(vertex_attrib(vertices), 0, 1);
    PixelColor: float4(1, 0, 0, 1);
};
```

Note that there should be no comma before the inline block when there are more blocks in the draw path:

```
draw DefaultShading, Sphere { PixelColor: Red; };
```

### Lambda blocks

The following draw statement applies DefaultShading specifies Scaling: 2.0f; in a lambda block, then applies a sphere, and finally overrides some meta properties in the inline block at the end:

```csharp
draw DefaultShading, { Scaling: 2.0f; }, Sphere { PixelColor: float4(1,0,0,1); };
```

Note that the lambda and inline blocks are slightly syntactically and semantically different:

* Lambda blocks are specified in-place for a block reference, and hence should be both preceeded and proceedeed by a comma.
* Lambda blocks are treated just like any other block declared elsewhere, with implicit using of the preceeding draw path. This means that any new meta properties must be declared public in the lambda block in order to be accessible for the ret of the draw statement.

### Scope variable capture

In inline and lambda blocks you may also reference fields, method arguments and local variables from the method scope direclty in meta property expressions. The following example draws ten spheres along the X-axis separated by 10.0 units.

```csharp
for (int i = 0; i < 10; i++)
    draw Sphere { Position: float3((float)i*10.0f, 0, 0); };
```
    
The following example draws a model from a COLLADA-file, using DefaultShading and a custom texture:

```csharp
draw Uno.Scenes.DefaultShading, Model("somemodel.DAE") { DiffuseMap: import Texture2D("sometexture.png"); };
```