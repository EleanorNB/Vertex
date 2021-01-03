Vertex Software Distribution
----------------------------
A minimal bootstrap system that rethinks the design of low-level software.

## Motivation
There's an engineering principle that I haven't ever seen written down anywhere, so I'll give it a name here:

**Layer Rigidly, Ice Freely.**

Meaning, if you expect something to be built upon, make it 100% sane, uniform, and predictable. This is almost universally ignored in software, due to the ease of sharing work and a long history of pioneers underestimating the future scope of the field. Common wisdom is that everything is so bad now that unfathomable bugs are inevitable and certain things are best left untouched. I believe we can remedy this by starting over from the beginning with the benefit of hindsight.

## Prior Art
My primary inspiration is AmigaOS. seL4, Redox, and of course POSIX are also significant influences. However, I will not aim for binary compatibility, or even pure source compatibility with any system. This is a wholly original project.

## License
I dread the day this becomes important. It's just a spec now anyway, and not even most of one -- if someone can actually realise my vision, then I consider my work done.

[Donate to the ZSF](https://github.com/sponsors/ziglang), I guess. I like them. This project's name was originally a play on theirs. 
