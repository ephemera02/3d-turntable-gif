# 🔄 3D Turntable GIF

Drop in a 3D model, get back a spinning GIF with a transparent background. That's the whole thing.

I kept needing turntable renders for my portfolio, and every tool I found was either a $400/year subscription, required more Blender than I've touched since I was 12 making Minecraft renders, or was some sketchy web app that wanted my email before it'd even show me a preview, so I built one myself with some help from Claude (because it knows how itself works better than I do)

One developer, one cat, one very patient AI. Open source 💜

> **Disclaimer:** This is my first skill. It serves a niche that probably doesn't exist, it's almost certainly fucking useless to 99% of people, and the code is ass. If you're here, you're either lost or you're exactly the kind of unhinged that I respect. Welcome.

## What It Does

You hand it an `.obj`, `.glb`, `.gltf`, or `.fbx` file, and it spins up a headless browser, renders your model in Three.js with studio lighting and a transparent background, takes a bunch of screenshots as it rotates, and stitches them into a looping GIF with ImageMagick. No GPU required, because it runs on SwiftShader (software rendering), so it works on basically anything with a pulse.

It's also a Claude skill, so if you install it, you can upload a model, say "make it spin," and Claude handles format detection, texture mapping, lighting, rendering, encoding, all of it, without you touching a single config file, because you literally just talk to it.

## Supported Formats

- **OBJ + MTL** is the workhorse. Always works and always looks right. Probably.
- **GLTF / GLB** is the modern standard; materials usually come pre-baked, so there's less to fuss with.
- **FBX 7000+** just works. Three.js reads these natively, no drama.
- **FBX 6100** auto-converts through Facebook's FBX2glTF because literally nothing else on earth can parse files from 2008. Not Three.js, not Blender, not Assimp. I lost hours of my life discovering this.

## The Pipeline

```
3D model file
    ↓ format detection + texture inspection
Local HTTP server + Three.js modules
    ↓ headless Chrome + Puppeteer
Frame capture (transparent PNG screenshots)
    ↓ ImageMagick
Looping GIF with transparency
```

The whole thing runs in about 30-60 seconds, depending on how many frames you want. No cloud services, no uploads, no accounts, nothing leaves your machine.

## Material Presets

- **Metallic**: low roughness, high metalness, environment reflections. Makes things look expensive. Good for hardware, product shots, anything you want to look like it costs more than it does.
- **Matte/Stylized**: preserves original materials and adds a subtle env map without fighting the art style. For cartoon assets, coins, stuff that already looks how it's supposed to look.
- **Glass/Translucent**: MeshPhysicalMaterial with transmission, IOR, and opacity. Logos, UI elements, anything you want to see through. It's pretty.
- **Clean white**: minimal metalness, low roughness, the "product floating on a white background" look that every Shopify store uses.

## Quality Settings

| | Frames | Speed | Size | When to use it |
|---|--------|-------|------|------|
| Preview | 48 | Fast | ~1 MB | Checking if the model even loaded right |
| Standard | 60 | Medium | ~1.5 MB | Good enough for most things, honestly |
| High | 120 | Medium | ~2.5 MB | Portfolio-ready, smooth rotation |
| Cinematic | 120 | Slow | ~2.5 MB | When you actually want people to stop scrolling and look |

## Installation (as a Claude Skill)

Clone this into your Claude skills directory:

```
/mnt/skills/user/3d-turntable-gif/
├── SKILL.md
└── references/
    ├── setup.md
    ├── formats.md
    └── capture-template.md
```

Or grab the `.skill` file from [Releases](../../releases) and install it directly.

Once it's in, upload a 3D model to Claude and say something like:
- "make a spinning GIF of this"
- "turntable render, transparent background, slow"
- "product spin animation"

and Claude figures out the rest.

## Requirements

- Node.js
- Puppeteer (headless Chrome)
- ImageMagick
- Three.js r160, downloaded locally per render (no CDN, because I learned that lesson the hard way too)

## Things I Learned The Hard Way So You Don't Have To

- CDN imports in headless Chrome cause navigation timeouts that will make you question your career choices, so download the Three.js modules locally unless you want to lose an evening to bullshit.
- FBX files from 2008 are not readable by anything made after 2015. Only Facebook's proprietary converter works, and only because it bundles the Autodesk SDK internally. I am still pissed about this.
- Headless Chrome drops the WebGL context during model loading, so you have to render like 10 throwaway frames after load so it can stabilize, otherwise your first real frames come out blank and you spend an hour debugging something that isn't actually broken. Ask me how I know. Actually don't.
- MTL files exported from Windows use backslash paths, and Unix hates that. So, you won't get an error message, you'll get invisible textures and a deep sense of "what the fuck is wrong with my model" until you open the `.mtl` in a text editor three hours later.
- When an OBJ has multiple materials, Three.js gives you an array instead of a single material. If you don't check for that, everything becomes one flat color, and you'll think your model is corrupted when it's fine.
- `-layers Optimize` in ImageMagick saves file size but absolutely shits on transparency quality, so don't use it on anything you care about.
- If you have both an FBX and an OBJ of the same model, use the OBJ every single time. I cannot stress this enough, and I will die on this hill.

## Credits

Written and built by Eph. Claude (Anthropic) "assisted", mostly by rendering an iPod model upside down approximately 47 times while I asked "why is it upside down" in increasingly creative ways. The Cat observed from a nearby surface and contributed nothing of value, as usual. Icons would be here if I'd made any.

🐾 Other projects: [InkPaw](https://github.com/ephemera02/InkPaw) · [SillyLoreSmith](https://github.com/ephemera02/SillyLoreSmith) · 💜 Support development: [ephemeradev.net](https://ephemeradev.net), tips welcome, never required

Created by Eph at [Ephemera](https://ephemeradev.net)

MIT License. See LICENSE for details.
