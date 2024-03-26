# Using Quilt Config Outside of Quilt Mods

Quilt Config was created for modding, but is designed to be used anywhere, even outside of mods!
This page will explain how to set up Quilt Config in a non-mod environment.
Remember you can still reference the other tutorials: 

## Setup

To use Quilt Config, we need to set up our own `ConfigEnvironment`. The environment serves two main purposes:
- Dictating *where* config files will be saved and loaded. When writing code for this, keep in mind the differences between filesystems on Windows, Linux, and MacOS, the main platforms for a desktop app.
- Providing serializers to define *how* config files will be saved and loaded. Note that 
