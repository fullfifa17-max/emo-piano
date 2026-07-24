-- EMO Piano Hub v1.0
-- dx9ware piano autoplayer — MIDI2LUA compatible
-- Paste raw MIDI2LUA code or URL directly into the hub
-- Play/Pause/Resume, Seek, Live BPM

if not _G.EMO_UI_LIB then _G.EMO_UI_LIB=loadstring(dx9.Get("https://raw.githubusercontent.com/fullfifa17-max/emo_UI/main/emo_ui.lua"))() end
local UI=_G.EMO_UI_LIB
local _sz=dx9.size()
local SX,SY=_sz.width,_sz.height
local DS=dx9.DrawString
local DFB=dx9.DrawFilledBox
local DB=dx9.DrawBox
local KP=dx9.keypress
local KR=dx9.keyrelease
local clock=os.clock
local floor=math.floor

local VK={
    ["1"]=0x31,["2"]=0x32,["3"]=0x33,["4"]=0x34,["5"]=0x35,
    ["6"]=0x36,["7"]=0x37,["8"]=0x38,["9"]=0x39,["0"]=0x30,
    ["q"]=0x51,["w"]=0x57,["e"]=0x45,["r"]=0x52,["t"]=0x54,
    ["y"]=0x59,["u"]=0x55,["i"]=0x49,["o"]=0x4F,["p"]=0x50,
    ["a"]=0x41,["s"]=0x53,["d"]=0x44,["f"]=0x46,["g"]=0x47,
    ["h"]=0x48,["j"]=0x4A,["k"]=0x4B,["l"]=0x4C,
    ["z"]=0x5A,["x"]=0x58,["c"]=0x43,["v"]=0x56,["b"]=0x42,
    [";"]=0xBA,["'"]=0xDE,[","]=0xBC,["."]=0xBE,["/"]=0xBF,
}
local VK_CTRL=0x11;local VK_SPACE=0x20;local NH=0.05
local INDEX_URL="https://raw.githubusercontent.com/juygtfdw2/emo-piano/main/songs/index.txt"

-- State persists across frames + sessions (while dx9 is open)
if not _G.EP then
    _G.EP={
        state="idle",events=nil,idx=0,total=0,nextT=0,
        name="",cache={},songs={},customSongs={},loaded=false,
        held={},seekPct=-1,lastInput="",rebuildUI=true,
    }
end
local P=_G.EP

-- Load song index from GitHub
if not P.loaded then
    pcall(function()
        local raw=dx9.Get(INDEX_URL)
        if raw and #raw>5 then
            P.songs={}
            for line in raw:gmatch("[^\r\n]+") do
                if line:sub(1,1)~="#" and #line>3 then
                    local name,artist,bpm,url=line:match("^(.-)|(.-)|(%d+)|(.+)$")
                    if name and url then P.songs[#P.songs+1]={name=name,artist=artist or "?",bpm=tonumber(bpm) or 120,url=url,src="lib"} end
                end
            end
        end
    end)
    P.loaded=true
end

-- ============================================================
-- PARSER
-- ============================================================
local function parseSong(raw)
    local events={};local bpm=120
    local bpmMatch=raw:match("bpm%s*=%s*(%d+)")
    if bpmMatch then bpm=tonumber(bpmMatch) end
    local beatSec=60/bpm;local pR=0
    for line in raw:gmatch("[^\r\n]+") do
        local s=line:match("^%s*(.-)%s*$")
        if s and s~="" and s:sub(1,2)~="--" then
            local rd=s:match("rest%(([%d%.]+)")
            if rd then pR=pR+(tonumber(rd)*beatSec) end
            local kp=s:match('keypress%("([^"]+)"')
            if kp then
                local ctrl=false;local key=kp
                if kp:sub(1,5)=="Ctrl+" then ctrl=true;key=kp:sub(6) end
                local vk=VK[key]
                if vk then events[#events+1]={d=pR,t="n",vk=vk,ctrl=ctrl};pR=0 end
            end
            if s:match("pedalDown%(") then events[#events+1]={d=pR,t="pd"};pR=0 end
            if s:match("pedalUp%(") then events[#events+1]={d=pR,t="pu"};pR=0 end
        end
    end
    local cumT=0
    for i=1,#events do cumT=cumT+events[i].d;events[i].cumT=cumT end
    return events,bpm,cumT
end

-- ============================================================
-- UI — rebuilt when custom songs are added
-- ============================================================
local T=UI.Theme({accent={180,100,255},title="EMO | Piano Hub",footerTxt="EMO | discord.gg/coi"})
local W=UI.Window({Index="PIANO_V6",Theme=T,Width=460,Height=400,ToggleKey="[INSERT]"})
local c1=W:Cat("Player")
local t1=c1:Tab("Songs")
local sL=t1:Card("Library","left")
local sR=t1:Card("Controls","right")

-- Build combined song list: library + custom
local allSongs={}
for i=1,#P.songs do allSongs[#allSongs+1]=P.songs[i] end
for i=1,#P.customSongs do allSongs[#allSongs+1]=P.customSongs[i] end
local songNames={}
for i=1,#allSongs do
    local tag=allSongs[i].src=="custom" and "[+] " or ""
    songNames[i]=tag..allSongs[i].name.." - "..allSongs[i].artist
end
if #songNames==0 then songNames={"No songs loaded"} end

sL:Dropdown("Song","song",songNames,songNames[1])
sL:Textbox("Paste Code/URL","input","")
sL:Textbox("Song Name","sname","Custom Song")

sR:Toggle("Play / Pause","go",false)
sR:Toggle("Stop","stop",false)
sR:Toggle("Load Pasted","load",false)
sR:Slider("BPM %","bpm",100,25,300,0)
sR:Slider("Seek %","seek",0,0,100,0)

local c2=W:Cat("Config")
c2:Tab("Settings"):Card("Settings","left"):Keybind("Menu Key","mk","[INSERT]")
local function V(k) return W:Val(k) end

-- ============================================================
-- LOAD PASTED CODE/URL
-- ============================================================
if V("load") then
    local input=V("input") or ""
    local sname=V("sname") or "Custom Song"
    if #input>10 then
        local raw=input
        local isURL=input:match("^https?://")
        if isURL then
            pcall(function() raw=dx9.Get(input) end)
        end
        if raw and #raw>10 then
            local events,bpm,totalTime=parseSong(raw)
            if #events>0 then
                local key="custom_"..#P.customSongs+1
                P.cache[key]={events=events,bpm=bpm,totalTime=totalTime}
                P.customSongs[#P.customSongs+1]={name=sname,artist="Custom",bpm=bpm,url=key,src="custom"}
                -- Song is now in cache and dropdown (next frame rebuild)
            end
        end
    end
end

-- ============================================================
-- LOAD SONG DATA
-- ============================================================
local function getSongData(songInfo)
    if P.cache[songInfo.url] then return P.cache[songInfo.url] end
    if songInfo.src=="custom" then return nil end
    local ok,raw=pcall(function() return dx9.Get(songInfo.url) end)
    if not ok or not raw or #raw<10 then return nil end
    local events,bpm,totalTime=parseSong(raw)
    if #events>0 then P.cache[songInfo.url]={events=events,bpm=bpm,totalTime=totalTime};return P.cache[songInfo.url] end
    return nil
end

local ALL_VK={0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x30,
    0x51,0x57,0x45,0x52,0x54,0x59,0x55,0x49,0x4F,0x50,
    0x41,0x53,0x44,0x46,0x47,0x48,0x4A,0x4B,0x4C,
    0x5A,0x58,0x43,0x56,0x42,0xBA,0xDE,VK_CTRL,VK_SPACE}
local function releaseAll()
    for i=1,#ALL_VK do pcall(function() KR(ALL_VK[i]) end) end
    P.held={}
end
local function fullStop() P.state="idle";P.events=nil;P.idx=0;P.total=0;P.name="";releaseAll() end

-- ============================================================
-- CONTROLS
-- ============================================================
if V("stop") then fullStop() end

local goOn=V("go")

if goOn and P.state=="idle" then
    local sel=V("song");local songInfo=nil
    for i=1,#allSongs do if songNames[i]==sel then songInfo=allSongs[i];break end end
    if songInfo then
        local data=getSongData(songInfo)
        if data and #data.events>0 then
            P.events=data.events;P.idx=1;P.total=#data.events
            P.name=songInfo.name;P.state="playing";P.held={};P.seekPct=-1
            P.nextT=clock()+(data.events[1].d*100/(V("bpm") or 100))
        end
    end
end

if not goOn and P.state=="playing" then P.state="paused";releaseAll() end
if goOn and P.state=="paused" then
    P.state="playing"
    if P.idx<=P.total then P.nextT=clock()+(P.events[P.idx].d*100/(V("bpm") or 100)) end
end

-- SEEK
if P.events and P.total>0 then
    local sv=V("seek") or 0
    if sv~=P.seekPct and P.seekPct~=-1 then
        releaseAll()
        local targetT=(P.events[P.total].cumT or 1)*(sv/100)
        local ni=1
        for i=1,P.total do if P.events[i].cumT>=targetT then ni=i;break end;ni=i end
        P.idx=ni
        if P.state=="playing" then P.nextT=clock()+(P.events[ni].d*100/(V("bpm") or 100)) end
    end
    P.seekPct=sv
end

-- ============================================================
-- PLAYBACK
-- ============================================================
if P.state=="playing" and P.events then
    local now=clock();local scale=100/(V("bpm") or 100);local batch=0
    local rm={}
    for i=#P.held,1,-1 do
        if now>=P.held[i].at then KR(P.held[i].vk);if P.held[i].ctrl then KR(VK_CTRL) end;rm[#rm+1]=i end
    end
    for i=1,#rm do table.remove(P.held,rm[i]) end
    while P.idx<=P.total and now>=P.nextT and batch<80 do
        local ev=P.events[P.idx]
        if ev.t=="n" then
            if ev.ctrl then KP(VK_CTRL) end
            KP(ev.vk)
            P.held[#P.held+1]={vk=ev.vk,ctrl=ev.ctrl or false,at=now+NH}
        elseif ev.t=="pd" then KP(VK_SPACE)
        elseif ev.t=="pu" then KR(VK_SPACE) end
        P.idx=P.idx+1;batch=batch+1
        if P.idx<=P.total then P.nextT=P.nextT+(P.events[P.idx].d*scale) end
    end
    if P.idx>P.total then
        for i=1,#P.held do KR(P.held[i].vk);if P.held[i].ctrl then KR(VK_CTRL) end end
        fullStop()
    end
end

-- ============================================================
-- HUD
-- ============================================================
local hx=SX/2-150;local hy=SY-55;local hW=300;local hH=42
DFB({hx,hy},{hx+hW,hy+hH},{20,10,30});DB({hx,hy},{hx+hW,hy+hH},{180,100,255})
if P.state~="idle" and P.total>0 then
    local pct=floor(P.idx/P.total*100)
    local st=P.state=="playing" and ">" or "||"
    DS({hx+5,hy+3},{255,255,255},st.." "..P.name)
    DS({hx+5,hy+17},{200,200,200},pct.."% | BPM:"..floor(V("bpm") or 100).."%")
    local bx=hx+5;local by=hy+31;local bW=hW-10;local filled=floor(bW*pct/100)
    DFB({bx,by},{bx+filled,by+6},{180,100,255});DB({bx,by},{bx+bW,by+6},{100,50,150})
    DFB({bx+filled-2,by-1},{bx+filled+2,by+7},{255,255,255})
else
    DS({hx+5,hy+5},{180,100,255},"EMO Piano Hub v5")
    DS({hx+5,hy+19},{150,150,150},#allSongs.." songs | Paste code & Load, or select & Play")
    DS({hx+5,hy+31},{100,100,100},"Custom: "..#P.customSongs.." loaded this session")
end
W:Draw()
