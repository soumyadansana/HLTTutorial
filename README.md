# HLTTutorial
Follow the HLT recipe for running on Summer21 MC https://twiki.cern.ch/twiki/bin/view/CMSPublic/SWGuideGlobalHLT
NOTE : Follow this recipe on lxplus because the different paths are easier to access. Import the HLT paths files etc in lxplus and then import to T2B cluster. ```setup.sh```
## Setup
Setup the release, import HLTrigger package, compile
```
cmsrel CMSSW_12_3_1
cd CMSSW_10_6_1_patch3/src
cmsenv
git cms-addpkg HLTrigger/Configuration
scram b -j 8
```
Follow steps mentioned in latest HLT recipe
Create a proxy (to access remote files) 

```
voms-proxy-init --voms cms --valid 168:00 
```

Clone this repository, and compile (N.B. some unused variables in the code introduce warnings)

```
git clone https://github.com/soumyadansana/HLTTutorial.git
USER_CXXFLAGS="-Wno-delete-non-virtual-dtor -Wno-error=unused-but-set-variable -Wno-error=unused-variable" scram b
```
Move to HLTrigger/Configuration/test

```
cd HLTrigger/Configuration/test
```

Now, fetch the needed trigger configuration for displaced dimuon triggers.
Note : This step needs to be performed in lxplus so that menus are directly accessible. Once the trigger cff file is made, can be copied to T2B and normally run.

```
hltGetConfiguration --cff --offline /dev/CMSSW_12_3_0/GRun --paths HLTriggerFirstPath,HLTriggerFinalPath --mc --eras Run3 --l1-emulator FullMC --l1 l1Menu_Collisions2022_v1_0_0_xml --globaltag auto:phase1_2021_realistic --unprescale > HLT_User_cff.py

hltGetConfiguration /users/sonawane/HLT_V2/HLT_V2_DDMpaths/V14 --cff --mc --globaltag auto:phase1_2021_realistic --unprescale --offline --l1-emulator FullMC --l1 L1Menu_Collisions2022_v1_0_0_xml >> HLT_User_cff.py
 ```
 You then need to modify HLT_User_cff.py : search for these two lines, they appear twice in the config file. Comment out their second appearance (not the first one). 
 ```
#import FWCore.ParameterSet.Config as cms                                                                                                                                                                   
#fragment = cms.ProcessFragment( "HLT" )
 ```
 
 We could also add HLT_IsoMu24 for prompt LLP decays (needs to be checked and instructions updated). 
```
hltGetConfiguration --cff --offline /dev/CMSSW_12_3_0/GRun --paths HLTriggerFirstPath, HLT_IsoMu24,HLTriggerFinalPath --mc --eras Run3 --l1-emulator FullMC --l1 l1Menu_Collisions2022_v1_0_0_xml --globaltag auto:phase1_2021_realistic --unprescale > HLT_User_cff.py

ltGetConfiguration /users/sonawane/HLT_V2/HLT_V2_DDMpaths/V14 --cff --mc --globaltag auto:phase1_2021_realistic --unprescale --offline --l1-emulator FullMC --l1 L1Menu_Collisions2022_v1_0_0_xml >> HLT_User_cff.py
```
Remove similar lines as before.
  
Copy the files to the python subdirectory
```
cp HLT_User_cff.py ../python/.
```

Produce a HLT2_HLT.py config file to be run with cmsRun HLT2_HLT.py. We will need to rerun HLT using RAW data and and then access offline information from MINIAOD. In order to do that, we need to specify the MINIAOD file as the main input file and the list of all its parent RAW files as secondary input files: 
```
cmsDriver.py --fileout file:temp.root --python_filename HLT2_cfg.py --conditions auto:phase1_2021_realistic --step HLT:User --geometry DB:Extended --filein file:root://cms-xrd-global.cern.ch//store/user/sdansana/HToSS/MC/MINIAODSIM/MINIAODSIM_NLO_ggH_HToSS_SmuonHadronFiltered_MH125_MS2_ctauS100_2021_220324/step4_3.root --secondfilein file:root://cms-xrd-global.cern.ch//store/user/sdansana/HToSS/MC/DIGI2RAW/DIGI2RAW_NLO_ggH_HToSS_SmuonHadronFiltered_MH125_MS2_ctauS100_2021_220322/step2_3.root --processName=HLT2 --era Run3 --no_exec --mc -n 100 --inputCommands=keep *,drop Run3Scouting*_hltScouting*Packer__HLT
```

So far the HLT2_cfg.py file contains a configuration to rerun HLT on top of RAW. 
We need to add our EDAnalyzer as a second module in the process. After: 
```
process.RECOSIMoutput_step = cms.EndPath(process.RECOSIMoutput)
```
Add: 
```
process.demo = cms.EDAnalyzer('TriggerAnalyzerRAWMiniAOD')
process.TFileService = cms.Service("TFileService",
                                   fileName = cms.string( "out.root" )
                                   )
process.demo_step = cms.EndPath(process.demo)
```
Replace the line:
```
process.schedule.extend([process.endjob_step,process.RECOSIMoutput_step])
```
by:
```
process.schedule.extend([process.endjob_step, process.demo_step])
```
N.B.: Keeping RECOSIMoutput_step in the process would create an updated (big!) RAW file 
with the information related to the trigger menu you reran. 
This file is typically quite big so we will drop it here. As an exercise, you may wish to produce 
it and take a look at its content, for example by doing: 
```
edmDumpEventContent --regex HLT temp.root
```
Test the code: 
```
cmsRun HLT2_cfg.py 
```
