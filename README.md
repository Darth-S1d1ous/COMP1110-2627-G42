# COMP1110-2627-G42

## Run the evacuation simulation

Commands below are run from `scenario/`. Paths in `most.evacuation.sumocfg` are relative to that directory. `sumo` and `sumo-gui` must be on `PATH` (Eclipse SUMO).

```powershell
cd "D:\hku\COMP 1110\COMP1110-2627-G42\scenario"
```

Headless run (full hour, writes `evacuation.tripinfo.xml`, `evacuation.summary.xml`, and `evacuation.sim.log`):

```powershell
sumo -c most.evacuation.sumocfg
```

GUI. `--delay 100` pauses 100 ms after each step so the window does not freeze and finish the hour immediately. `--end 900` stops after 15 simulated minutes, which covers the 10-minute departure window.

```powershell
sumo-gui -c most.evacuation.sumocfg --delay 100 --end 900
```

After the window opens, use Locate, choose Edge, and enter `152886#1` to jump to the Stade Louis II departure edge. Then press play.