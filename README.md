# nusonic-minimal-example

This repository contains fhicl files for running a minimal inference request to an [NVIDIA Triton server](https://developer.nvidia.com/triton-inference-server) through LArSoft. The instructions below assume you have access to Fermilab's `dunegpvm` machines, or some other nodes with access to `mrb` and `ups`. 

The fhicl files here are meant to be copied into a local LArSoft development area. There is no installation of `nusonic-minimal-example` required, but you will need to [create a local LArSoft area](https://larsoft.org/setting-up-and-running-larsoft/) that includes [larrecodnn](https://github.com/LArSoft/larrecodnn), a repository housing LArSoft modules and fhicl files for running the [ProtoDUNE CVN](https://larsoft.org/cnn-based/) pixel-wise classification model. Furthermore, you'll need to generate an artroot file containing simulated particle interactions and set up a Triton server instance. The instructions below provide an example of how to do so.

# Setup
To create a local LArSoft development area, follow the instructions below on a machine with access to `mrb` and `ups`. 
```
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh # Or whichever experiment setup script you need
cd /some/clean/directory
mrb newDev v09_88_00 # LArSoft v09_88_00 area
source localProducts_larsoft_v09_88_00_e26_prof/setup
cd srcs/
mrb g -t v09_22_02 larrecodnn # Tag corresponding to larsoft v09_88_00
cd $MRB_BUILDDIR
mrbsetenv
```
The specific versions listed above can be changed depending on your needs. To view available LArSoft versions, run
```
ups list -aK+ larsoft
```
Then, to see which version of `larrecodnn` corresponds to that LArSoft version,
```
ups depend larsoft <version> -q <qualifiers> | grep larrecodnn
```
You can then repeat the instructions above with those version numbers. 

Next, copy the fcl files under the `fcl/` directory in this repository to your `larrecodnn` area. We recommend copying the files to the same directories as their existing analogues in `larrecodnn`. From the top directory of this repository, you can run
```
cp fcl/nusonic_imagepatternalgs.fcl $MRB_TOP/srcs/larrecodnn/larrecodnn/ImagePatternAlgs/Modules
cp fcl/nusonic_em_trk_job.fcl $MRB_TOP/srcs/larrecodnn/larrecodnn/ImagePatternAlgs/job
cp fcl/nusonic_dump_events_protodune_cnn.fcl $MRB_TOP/srcs/larrecodnn/larrecodnn/ImagePatternAlgs/job
```
Finally, build your local area by running
```
cd $MRB_BUILDDIR
mrb i -j 16 --generator ninja # Adjust the number of cores depending on your machine
```

# Generating an Input artroot File
If you're familiar with generating artroot files using a chain of fhicl files, you can skip this section. The following commands will generate an artroot file containing five single-particle events using ProtoDUNE geometry. There are many other combinations of fhicl files that will accomplish something similar; feel free to use whatever works for your needs.
```
setup dunesw v09_88_00d00 -q e26:prof # Or whatever version corresponds to your larsoft build
lar -c protoDUNE_gensingle.fcl -n 5
lar -c protoDUNE_refactored_g4.fcl -s gensingle_protoDUNE.root
lar -c protoDUNE_refactored_g4_stage2.fcl -s gensingle_protoDUNE_g4.root
lar -c protoDUNE_refactored_detsim.fcl -s gensingle_protoDUNE_g4_g4_stage2.root
lar -c protoDUNE_refactored_reco_stage1.fcl -s gensingle_protoDUNE_g4_g4_stage2_detsim.root
lar -c protoDUNE_refactored_reco_stage2.fcl -s gensingle_protoDUNE_g4_g4_stage2_detsim_reco1.root
```
This will generate an output file called `gensingle_protoDUNE_g4_g4_stage2_detsim_reco1_reco2.root`. 

# Setting up a Triton Server Instance
To create a Triton server instance, you can follow the instructions in the [Sonic tutorial page](https://yongbinfeng.gitbook.io/sonictutorial/prerequisites#run-with-singularity-apptainer). When running on a multi-user system like a gpvm, it's recommended to use Apptainer (formerly Singularity) instead of Docker since Docker effectively grants root permissions. For the purpose of this example, we'll simply use an available model on `ailab01.fnal.gov` under the `/models` area:
```
singularity run --nv -B /models/cnn_emtrkmichel_1:/models/cnn_emtrkmichel_1:ro \
            triton_21.12.sif tritonserver --model-repository=/models
```
You should then see an output message showing a `gRPC` process running on port 8001. To access this server from another node, such as a gpvm, you'll need to setup an ssh tunnel by running
```
ssh -4 -NL 8001:localhost:8001 $USER@ailab01.fnal.gov
```
from your desired node. Note that the `-4` here means we're using IPv4 instead of IPv6, which may not be necessary for every use case. 

# Sending an Inference Request to the Server
Finally, after all the above pieces are put together, you can run
```
lar -c nusonic_em_trk_job.fcl -s /path/to/gensingle_protoDUNE_g4_g4_stage2_detsim_reco1_reco2.root
```
which will produce an output numpy file called `output0.npy`. 

# Analyzing the Output
Uh...TBD.

# Further Reading
The instructions in this repository were largely sourced from the following links, plus some old-fashioned, brute-force trial and error. 
- [SONIC tutorial](https://yongbinfeng.gitbook.io/sonictutorial/prerequisites)
- [SonicCore](https://github.com/cms-sw/cmssw/tree/master/HeterogeneousCore/SonicCore) and [SonicTriton](https://github.com/cms-sw/cmssw/tree/master/HeterogeneousCore/SonicTriton) READMEs
- [GPU as-a-service in LArSoft tutorial](https://larsoft.github.io/LArSoftWiki/GPU_as_a_Service)
- [Intro to art and LArSoft](https://dune.github.io/computing-training-basics-short/04-intro-art-larsoft/index.html)

# Contributing
If you encounter a problem with the instructions in this repository, please [create an issue](https://docs.github.com/en/issues/tracking-your-work-with-issues/creating-an-issue) describing the problem, including step-by-step instructions for reproducing it. 
