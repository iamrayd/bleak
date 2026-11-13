# Setup before the first agent runs

Everything a human must do, in order. Parts A and B block the first task.
Part C blocks tasks F7 and F8. Part D can wait until the task named.

Each step ends with a check. Do not start the agents until every check in
parts A to C passes.

## A. Install

1. **Git.** Check: `git --version`.
2. **Roblox Studio.** Download from https://create.roblox.com, install, and
   log in with the account that will own the game.
   Check: Studio opens and shows your account name.
3. **Python 3.11 or later.** Download from https://www.python.org/downloads/.
   On Windows, tick "Add python.exe to PATH" in the installer.
   Check: open a new terminal and run `python --version`.
4. **Rokit.** Download the Windows build from
   https://github.com/rojo-rbx/rokit/releases and follow the install steps in
   the Rokit README. Open a new terminal afterwards.
   Check: `rokit --version`.

The agent installs Rojo, Selene, StyLua and luau-lsp itself through Rokit
(plan 01, task F1). You do not install those.

GitHub CLI (`gh`) is optional. The plan only needs `git push`.

## B. Roblox experiences

5. In the Creator Dashboard (https://create.roblox.com/dashboard/creations),
   create two experiences from the Baseplate template:
   - **Bleak** — the real game.
   - **Bleak Test** — used only by automated tests. Keep it private.
6. For **Bleak Test**, copy the Universe ID and the Start Place ID from the
   experience's menu on the dashboard. Do the same for **Bleak** and keep
   them for later.
7. Set the test ids as user environment variables. In PowerShell:

   ```powershell
   setx BLEAK_TEST_UNIVERSE_ID "<universe id of Bleak Test>"
   setx BLEAK_TEST_PLACE_ID "<start place id of Bleak Test>"
   ```

   Check: open a **new** terminal and run `echo $env:BLEAK_TEST_UNIVERSE_ID`.

## C. Keys and Studio connection

8. **Open Cloud API key.** Go to
   https://create.roblox.com/dashboard/credentials and create an API key.
   - Add the **Bleak Test** experience to it.
   - Give it the scopes `universe.place:write` and
     `universe.place.luau-execution-session:write`.
   - If the page lists the permissions under different names, pick the ones
     for "write place" and "Luau execution session: write", and tell the
     agent the names you saw.
   - Copy the key once. Roblox does not show it again.

   ```powershell
   setx ROBLOX_API_KEY "<the key>"
   ```

   Check: a new terminal prints a value for `echo $env:ROBLOX_API_KEY.Length`.

   Never paste the key into a file in this repository or into a chat.
9. **GitHub Actions.** In the repository on GitHub, under
   Settings > Secrets and variables > Actions:
   - Secret: `ROBLOX_API_KEY`.
   - Variables: `BLEAK_TEST_UNIVERSE_ID`, `BLEAK_TEST_PLACE_ID`.
10. **Rojo Studio plugin.** In Studio, open the Creator Store (Toolbox),
    search for "Rojo" by the Rojo publisher, and install it. Its version must
    be 7.7.x to match the Rojo 7.7.1 command-line tool; if the plugin and the
    tool disagree, the Connect button reports a version mismatch.
    Check: a Rojo button appears in the Plugins tab.
11. **Studio MCP server.** In Studio: Assistant > … > Manage MCP Servers >
    turn on "Enable Studio as MCP server".
12. **Connect Claude Code to Studio.** In a terminal in this repository:

    ```
    claude mcp add roblox-studio -- cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat
    ```

    Check: `claude mcp list` shows `roblox-studio` as connected while Studio
    is open. If the path or menu differs, follow
    https://create.roblox.com/docs/studio/mcp.

## D. Later, when the task asks

| When | Step |
|------|------|
| Whenever an agent works in Studio | Open the **Bleak** place in Studio, run `rojo serve default.project.json`, press Connect in the Rojo plugin. Keep Studio open |
| Before task E9 | In Studio Properties for `Lighting`: set `LightingStyle` to `Realistic` and turn on `PrioritizeLightingQuality` |
| Before task A2 | Check whether your account is ID-verified (audio upload quota). Approve the audio source list the agent shows you |
| Before task C14 | In Studio: Game Settings > Security > turn on "Enable Studio Access to API Services" |
| End of each milestone | In Studio: File > Save to File as `place/Bleak.rbxl` |
| Before task Q12 | Gather 2–4 people for the multi-client playtest |
| Release | Publish from Studio, fill in the content maturity questionnaire, set Max Players to 4, choose the voice chat setting, make the experience public |

## How this maps to the plan

| Setup step | Human gate in the master plan |
|------------|-------------------------------|
| 1–4 | H-1 |
| 5–7 | H-2 |
| 8–9 | H-3 |
| 10–12 | H-4 |
| Part D | H-5 to H-9 |
