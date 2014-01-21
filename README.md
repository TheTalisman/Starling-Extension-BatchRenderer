Batch Renderer Starling Extension
================================

Ever wanted to create a custom DisplayObject? Needed to render a non-rectangular geometry? Had to pass custom data via vertex atribute (va) registers to your shader? Cried inside (just a little) when custom texture processing was necessary? 

If so, I might have something just for you. Behold the Batch Renderer!

What is Batch Renderer?
-----------------------

Batch Renderer is an extension for Starling Framework - a GPU powered, 2D rendering framework. In Starling, all rendering is (mostly) done using Quad classes which, when added to the Starling's display list hierarchy, render a rectangular region onto the screen. But sometimes you want to do something than this and for that, you can use the BatchRenderer class.

First subclass it, like so:
```as3
use namespace renderer_internal;

public class TexturedGeometryRenderer extends BatchRenderer {
    public static const POSITION:String         = "position";
    public static const UV:String               = "uv";

    public static const INPUT_TEXTURE:String    = "inputTexture";

    private var _positionID:int, _uvID:int;

    // shader variables
    private var uv:IRegister = VARYING[0];  // v0 is used to pass interpolated uv from vertex to fragment shader

    public function TexturedGeometryRenderer() {
        setVertexFormat(createVertexFormat());
    }

    public function get inputTexture():Texture { return getInputTexture(INPUT_TEXTURE); }
    public function set inputTexture(value:Texture):void { setInputTexture(INPUT_TEXTURE, value); }

    public function getVertexPosition(vertex:int, position:Vector.<Number> = null):Vector.<Number> { return getVertexData(vertex, _positionID, position); }
    public function setVertexPosition(vertex:int, x:Number, y:Number):void { setVertexData(vertex, _positionID, x, y); }

    public function getVertexUV(vertex:int, uv:Vector.<Number> = null):Vector.<Number> { return getVertexData(vertex, _uvID, uv); }
    public function setVertexUV(vertex:int, u:Number, v:Number):void { setVertexData(vertex, _uvID, u, v); }

    override protected function vertexShaderCode():void {
        comment("output vertex position");
        multiply4x4(OUTPUT, getVertexAttribute(POSITION), getRegisterConstant(PROJECTION_MATRIX));

        comment("pass uv to fragment shader");
        move(uv, getVertexAttribute(UV));
    }

    override protected function fragmentShaderCode():void {
        var input:ISampler = getTextureSampler(INPUT_TEXTURE);

        comment("sample the texture and send resulting color to the output");
        sampleTexture(OUTPUT, uv, input, [TextureFlag.TYPE_2D, TextureFlag.MODE_CLAMP, TextureFlag.FILTER_LINEAR, TextureFlag.MIP_NONE]);
    }

    private function createVertexFormat():VertexFormat {
        var format:VertexFormat = new VertexFormat();

        _positionID = format.addProperty(POSITION, 2);  // x, y; id: 0
        _uvID       = format.addProperty(UV, 2);        // u, v; id: 1

        return format;
    }
}

```

Doesn't look that scary, does it? Let's have a look at it in details.

First, a VertexFormat is created and set:
```as3
public static const POSITION:String         = "position";
public static const UV:String               = "uv";
//...
private var _positionID:int, _uvID:int;
//...
public function TexturedGeometryRenderer() {
    setVertexFormat(createVertexFormat());
}
//...
private function createVertexFormat():VertexFormat {
    var format:VertexFormat = new VertexFormat();

    _positionID = format.addProperty(POSITION, 2);  // x, y; id: 0
    _uvID       = format.addProperty(UV, 2);        // u, v; id: 1

    return format;
}

```

Vertex format is crucial - it tells the BatchRenderer implementation how and what different kinds of data are going to store data in each vertex. With this TexturedGeometryRenderer each vertex stores two kinds of data: vertex position in 2D space (x, y) and texture mapping coords (u, v). Also notice, each kind of data, when added to VertexFormat (by addProperty() method) is registered with a unique name (here "position" and "uv", passed via static constants) and once registered is given an unique id (stored in '_positionID' and '_uvID'). The former can be used in when writing shaders' code and the later is useful for fast accessing each property in AS3 code.

