## Angora

https://hub.docker.com/r/zjuchenyuan/angora

Source: https://github.com/AngoraFuzzer/Angora

```
Current Version: 1.2.2
More Versions: Ubuntu 16.04, LLVM 7.0.0, Rust 1.37.0, PIN 3.7, golang 1.6.2
Last Update: 2019/09 Active
Language: Rust, C++ (DataFlow Sanitizer)
Type: Mutation Fuzzer, Compile-time Instrumentation
Tags: Context-sensitive branch coverage, Byte-level taint tracking (modified DFSan), Search based on gradient descent, Type and shape inference, Input length exploration
```

## Guidance

Fuzzing MP3Gain 1.6.2 as an example.

### Step1: System configuration

Please refer to [AFL Guidance](https://hub.docker.com/r/zjuchenyuan/afl). 

### Step2: Compile target programs

```
mkdir -p $WORKDIR/example/build/mp3gain/angora/{fast,taint}

# build angora fast and taint binaries
cd $WORKDIR/example/code/mp3gain1.6.2
docker run --rm -w /work -it -v `pwd`:/work --privileged --env CC=/angora/bin/angora-clang --env CXX=/angora/bin/angora-clang++ zjuchenyuan/angora sh -c "make clean; make"
mv mp3gain $WORKDIR/example/build/mp3gain/angora/fast/
docker run --rm -w /work -it -v `pwd`:/work --privileged --env CC=/angora/bin/angora-clang --env CXX=/angora/bin/angora-clang++ --env USE_TRACK=1 --env ANGORA_TAINT_RULE_LIST=/tmp/abilist.txt zjuchenyuan/angora sh -c "make clean; /angora/tools/gen_library_abilist.sh  /usr/lib/x86_64-linux-gnu/libmpg123.so discard > /tmp/abilist.txt; make"
mv mp3gain $WORKDIR/example/build/mp3gain/angora/taint/
```

### Step3: Start Fuzzing

```
cd $WORKDIR/example
rm -rf output/angora
docker run --rm -w /work -it -v `pwd`:/work --privileged zjuchenyuan/angora \
    /angora/angora_fuzzer --input seed/mp3 --output output/angora -t /work/build/mp3gain/angora/taint/mp3gain -- /work/build/mp3gain/angora/fast/mp3gain @@
```

### Explanation

#### ABI list for build tainted binary

DataFlow Sanitizer needs all code to be present, or provide an ABI list to tell how to propagate the tags for unknown functions.

Angora provides us `/angora/tools/gen_library_abilist.sh` to generate needed ABI list file, and we need to specify the file location via environment variable `ANGORA_TAINT_RULE_LIST`.

For more details, see https://github.com/AngoraFuzzer/Angora/blob/master/docs/build_target.md Model an external library.

#### Input Length

In [common/src/config.rs](https://github.com/AngoraFuzzer/Angora/blob/a3b25de4b1d68584d3027c0a0aa3da93bb571959/common/src/config.rs)

```
pub const MAX_INPUT_LEN: usize = 15000;
```

Angora only accept seed files less than 15000 bytes, so we changed `MAX_INPUT_LEN` in UNIFUZZ experiments to make fuzzers use same seed sets for a fair comparison.