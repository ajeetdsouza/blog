---
title: "Beating C with 70 Lines of Go"
description: "Writing a faster wc utility in Go"
date: 2019-11-20T08:09:39+05:30
type: "post"
---

Chris Penner's recent article, [Beating C with 80 Lines of Haskell](https://chrispenner.ca/posts/wc), has generated quite some controversy over the Internet, and it has since turned into a game of trying to take on the venerable `wc` with different languages:

- [Ada](http://verisimilitudes.net/2019-11-11)
- [C](https://github.com/expr-fi/fastlwc/)
- [Common Lisp](http://verisimilitudes.net/2019-11-12)
- [Dyalog APL](https://ummaycoc.github.io/wc.apl/)
- [Futhark](https://futhark-lang.org/blog/2019-10-25-beating-c-with-futhark-on-gpu.html)
- [Haskell](https://chrispenner.ca/posts/wc)
- [Rust](https://medium.com/@martinmroz/beating-c-with-120-lines-of-rust-wc-a0db679fe920)

Today, we will be pitting [Go](http://golang.org/) against `wc`. Being a compiled language with excellent concurrency primitives, it should be trivial to achieve [comparable performance](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/go-gcc.html) to C.

Note that while `wc` is also designed to read from `stdin`, handle non-ASCII text encodings, and parse command line flags ([manpage](https://ss64.com/osx/wc.html)), we will not be doing that here. Instead, like the articles mentioned above, we will focus on keeping our implementation as simple as possible.

## Benchmarking & comparison

We will use the GNU [time](https://www.gnu.org/software/time/) utility to compare elapsed time and maximum resident set size.

```sh
$ /usr/bin/time -f "%es %MKB" wc test.txt
```

We will use the [same version of wc](https://opensource.apple.com/source/text_cmds/text_cmds-68/wc/wc.c.auto.html) as the original article, compiled with `gcc 9.2.1` and `-O3`. For our own implementation, we will use `go 1.13.4` (I did try `gccgo` too, but the results were not very promising). We will run all benchmarks with the following setup:

- Intel Core i5-6200U @ 2.30 GHz (2 physical cores, 4 threads)
- 4+4 GB RAM @ 2133 MHz
- 240 GB M.2 SSD
- Fedora 31

For a fair comparison, all implementations will use a 16 KB buffer for reading input. The input will be two `us-ascii` encoded text files of 100 MB and 1 GB.

## A naïve approach

Parsing arguments is easy, since we only require the file path:

```go
if len(os.Args) < 2 {
    panic("no file path specified")
}
filePath := os.Args[1]

file, err := os.Open(filePath)
if err != nil {
    panic(err)
}
defer file.Close()
```

We're going to iterate through the text bytewise, keeping track of state. Fortunately, in this case, we require only 2 states:

- The previous byte was whitespace
- The previous byte was not whitespace

When going from a whitespace character to a non-whitespace character, we increment the word counter. This approach allows us to read directly from a byte stream, keeping memory consumption low.

```go
const bufferSize = 16 * 1024
reader := bufio.NewReaderSize(file, bufferSize)

lineCount := 0
wordCount := 0
byteCount := 0

prevByteIsSpace := true
for {
    b, err := reader.ReadByte()
    if err != nil {
        if err == io.EOF {
            break
        } else {
            panic(err)
        }
    }

    byteCount++

    switch b {
    case '\n':
        lineCount++
        prevByteIsSpace = true
    case ' ', '\t', '\r', '\v', '\f':
        prevByteIsSpace = true
    default:
        if prevByteIsSpace {
            wordCount++
            prevByteIsSpace = false
        }
    }
}

fmt.Printf("%d %d %d %s\n", lineCount, wordCount, byteCount, file.Name())
```

Let's run this:

|            | input size | elapsed time | max memory |
|:---------- | ----------:| ------------:| ----------:|
| `wc`       |     100 MB |       0.58 s |    2052 KB |
| `wc-naive` |     100 MB |       0.90s  |    1780 KB |
| `wc`       |       1 GB |       5.56 s |    2036 KB |
| `wc-naive` |       1 GB |       8.78 s |    1736 KB |

The good news is that our first attempt has already landed us pretty close to C in terms of performance. In fact, we're actually doing _better_ in terms of memory usage!

## Splitting the input

While buffering I/O reads is critical to improving performance, calling `ReadByte()` and checking for errors in a loop introduces a lot of unnecessary overhead. We can avoid this by manually buffering our read calls, rather than relying on `bufio.Reader`.

To do this, we will split our input into buffered chunks that can be processed individually. Fortunately, to process a chunk, the only thing we need to know about the previous chunk (as we saw earlier) is if its last character was whitespace.

Let's write a few utility functions:

```go
type Chunk struct {
    PrevCharIsSpace bool
    Buffer          []byte
}

type Count struct {
    LineCount int
    WordCount int
}

func GetCount(chunk Chunk) Count {
    count := Count{}

    prevCharIsSpace := chunk.PrevCharIsSpace
    for _, b := range chunk.Buffer {
        switch b {
        case '\n':
            count.LineCount++
            prevCharIsSpace = true
        case ' ', '\t', '\r', '\v', '\f':
            prevCharIsSpace = true
        default:
            if prevCharIsSpace {
                prevCharIsSpace = false
                count.WordCount++
            }
        }
    }

    return count
}

func IsSpace(b byte) bool {
    return b == ' ' || b == '\t' || b == '\n' || b == '\r' || b == '\v' || b == '\f'
}
```

Now, we can split the input into `Chunk`s and feed them to the `GetCount` function.

```go
totalCount := Count{}
lastCharIsSpace := true

const bufferSize = 16 * 1024
buffer := make([]byte, bufferSize)

for {
    bytes, err := file.Read(buffer)
    if err != nil {
        if err == io.EOF {
            break
        } else {
            panic(err)
        }
    }

    count := GetCount(Chunk{lastCharIsSpace, buffer[:bytes]})
    lastCharIsSpace = IsSpace(buffer[bytes-1])

    totalCount.LineCount += count.LineCount
    totalCount.WordCount += count.WordCount
}
```

To obtain the byte count, we can make one system call to query the file size:

```go
fileStat, err := file.Stat()
if err != nil {
    panic(err)
}
byteCount := fileStat.Size()
```

Now that we're done, let's see how this performs:

|             | input size | elapsed time | max memory |
|:----------- | ----------:| ------------:| ----------:|
| `wc`        |     100 MB |       0.58 s |    2052 KB |
| `wc-chunks` |     100 MB |       0.38 s |    1864 KB |
| `wc`        |       1 GB |       5.56 s |    2036 KB |
| `wc-chunks` |       1 GB |       3.72 s |    1864 KB |

Looks like we've blown past `wc` on both counts, and we haven't even begun to parallelize our program yet. [`tokei`](https://github.com/XAMPPRocky/tokei) reports that this program is just 70 lines of code!

## Parallelization

Admittedly, a parallel `wc` is overkill, but let's see how far we can go. The original article reads from the input file in parallel, and while it improved runtime, the author does admit that performance gains due to parallel reads might be limited to only certain kinds of storage, and would be detrimental elsewhere.

For our implementation, we want our code to be performant on _all_ devices, so we will not be doing this. We will set up 2 channels, `chunks` and `counts`. Each worker will read and process data from `chunks` until the channel is closed, and then write the result to `counts`.

```go
func ChunkCounter(chunks <-chan Chunk, counts chan<- Count) {
    totalCount := Count{}
    for {
        chunk, ok := <-chunks
        if !ok {
            break
        }
        count := GetCount(chunk)
        totalCount.LineCount += count.LineCount
        totalCount.WordCount += count.WordCount
    }
    counts <- totalCount
}
```

We will spawn one worker per logical CPU core:

```go
numWorkers := runtime.NumCPU()
runtime.GOMAXPROCS(numWorkers)

chunks := make(chan Chunk)
counts := make(chan Count)

for i := 0; i < numWorkers; i++ {
    go ChunkCounter(chunks, counts)
}
```

Now, we run in a loop, reading from the disk and assigning jobs to each worker:

```go
const bufferSize = 16 * 1024
lastCharIsSpace := true

for {
    buffer := make([]byte, bufferSize)
    bytes, err := file.Read(buffer)
    if err != nil {
        if err == io.EOF {
            break
        } else {
            panic(err)
        }
    }
    chunks <- Chunk{lastCharIsSpace, buffer[:bytes]}
    lastCharIsSpace = IsSpace(buffer[bytes-1])
}
close(chunks)
```

Once this is done, we can simply sum up the counts from each worker:

```go
totalCount := Count{}
for i := 0; i < numWorkers; i++ {
    count := <-counts
    totalCount.LineCount += count.LineCount
    totalCount.WordCount += count.WordCount
}
close(counts)
```

Let's run this and see how it compares to the previous results:

|              | input size | elapsed time | max memory |
|:------------ | ----------:| ------------:| ----------:|
| `wc`         |     100 MB |       0.58 s |    2052 KB |
| `wc-channel` |     100 MB |       0.21 s |    6632 KB |
| `wc`         |       1 GB |       5.56 s |    2036 KB |
| `wc-channel` |       1 GB |       2.10 s |    7520 KB |

Our `wc` is now a lot faster, but there has been quite a regression in memory usage. In particular, notice how our input loop allocates memory at every iteration! Channels are a great abstraction over sharing memory, but for some use cases, simply _not_ using channels can improve performance tremendously.

## Better parallelization

In this section, we will allow every worker to read from the file, and use `sync.Mutex` to ensure that reads don't happen simultaneously. We can create a new `struct` to handle this for us:

```go
type FileReader struct {
    File            *os.File
    LastCharIsSpace bool
    sync.Mutex
}

func (fileReader *FileReader) ReadChunk(buffer []byte) (Chunk, error) {
    fileReader.Lock()
    defer fileReader.Unlock()

    bytes, err := fileReader.File.Read(buffer)
    if err != nil {
        return Chunk{}, err
    }

    chunk := Chunk{fileReader.LastCharIsSpace, buffer[:bytes]}
    fileReader.LastCharIsSpace = IsSpace(buffer[bytes-1])

    return chunk, nil
}
```

We then rewrite our worker function to read directly from the file:

```go
func FileReaderCounter(fileReader *FileReader, counts chan Count) {
    const bufferSize = 16 * 1024
    buffer := make([]byte, bufferSize)

    totalCount := Count{}

    for {
        chunk, err := fileReader.ReadChunk(buffer)
        if err != nil {
            if err == io.EOF {
                break
            } else {
                panic(err)
            }
        }
        count := GetCount(chunk)
        totalCount.LineCount += count.LineCount
        totalCount.WordCount += count.WordCount
    }

    counts <- totalCount
}
```

Like earlier, we can now spawn these workers, one per CPU core:

```go
fileReader := &FileReader{
    File:            file,
    LastCharIsSpace: true,
}
counts := make(chan Count)

for i := 0; i < numWorkers; i++ {
    go FileReaderCounter(fileReader, counts)
}

totalCount := Count{}
for i := 0; i < numWorkers; i++ {
    count := <-counts
    totalCount.LineCount += count.LineCount
    totalCount.WordCount += count.WordCount
}
close(counts)
```

Let's see how this performs:

|            | input size | elapsed time | max memory |
|:---------- | ----------:| ------------:| ----------:|
| `wc`       |     100 MB |       0.58 s |    2052 KB |
| `wc-mutex` |     100 MB |       0.15 s |    2048 KB |
| `wc`       |       1 GB |       5.56 s |    2036 KB |
| `wc-mutex` |       1 GB |       1.53 s |    2052 KB |

Our parallelized implementation runs at more than 3.5x the speed of `wc`, while matching its memory consumption! This is pretty significant, especially if you consider that Go is a garbage collected language.

## Conclusion

While in no way does this article imply that Go > C, I hope it demonstrates that Go can be a viable alternative to C as a systems programming language.

If you have suggestions, questions, or complaints, feel free to [drop me a mail](mailto:98ajeet@gmail.com)!
