# Slide Launcher

Slide Launcher is a tiny Windows launcher/updater for the Slide desktop app.

The main Slide desktop receiver lives in the sibling `slide-desktop` project and is packaged as `Slide.jar`. This launcher checks whether a newer `Slide.jar` is available from the maintenance server, downloads it when needed, then starts it with the locally installed Java Runtime.

## What it does

- Checks for internet access before attempting an update.
- Downloads a remote version file.
- Compares the remote version with the local `version` file.
- Downloads `Slide.jar` when no local version exists or when the versions differ.
- Starts `Slide.jar` with `java.exe`.
- Hides the Java console window when launching the desktop app.

If the machine is offline or an update check fails, the launcher falls back to starting the local `Slide.jar` if it is present.

## How it fits into Slide

Slide is split into three projects:

- `slide-android`: Android app that captures touch, S Pen, gestures, and keyboard events.
- `slide-desktop`: desktop Java/Scala app that receives those events and emulates mouse/keyboard input.
- `SlideLauncher`: this small Windows bootstrapper for keeping and starting the desktop JAR.

This project does not emulate input by itself. Its only job is to keep the desktop app up to date and launch it.

## Runtime behavior

At startup, `main.go` does roughly this:

1. Try to open a TCP connection to `google.com:80` as a simple internet check.
2. If offline, call `runSlide()` immediately.
3. Fetch the remote version from:

   `http://lkuich.com/projects/slide/maint/version`

4. Read the local `version` file from the current working directory.
5. If the local file is missing or its contents do not match the remote version:
   - download the remote `version` file
   - download `Slide.jar` from:

     `http://lkuich.com/projects/slide/maint/Slide.jar`

6. Locate Java through the Windows registry key:

   `SOFTWARE\JavaSoft\Java Runtime Environment`

7. Start:

   `java.exe -jar Slide.jar`

The launcher expects `Slide.jar` and `version` to live in the same working directory as the executable.

## Important implementation details

- The code is Windows-specific.
- It reads the Java Runtime location from the Windows registry.
- It imports `golang.org/x/sys/windows/registry`, so it will not build for non-Windows targets without changes.
- It uses plain HTTP URLs for update downloads because this is old archival code.
- It has no UI; it silently downloads/updates and then starts the JAR.

## Building

This predates Go modules and contains a single `main.go` file.

A historical GOPATH-style build would look like:

```bash
go get golang.org/x/sys/windows/registry
go build -o SlideLauncher.exe main.go
```

For a modern module-based build, initialize a module first:

```bash
go mod init slide-launcher
go get golang.org/x/sys/windows/registry
go build -o SlideLauncher.exe .
```

Build this on Windows, or cross-compile with a Windows target:

```bash
GOOS=windows GOARCH=amd64 go build -o SlideLauncher.exe .
```

## Running

Place `SlideLauncher.exe` in the directory where you want `Slide.jar` and `version` to be stored, then run it.

On the first successful online run, it should download both files. On later runs, it will only replace them when the remote version changes.

## Known limitations

- The update endpoint is hard-coded.
- Downloads are unsigned and not integrity-checked.
- Java discovery assumes the legacy Oracle/Sun Java Runtime registry layout.
- Error handling is minimal; failures usually fall through to trying the local `Slide.jar`.
- The launcher uses `cmd.Output()`, so the Java process is started and waited on rather than detached.

## License

No separate license file is present in this folder. See the related Slide projects for the original project licensing context.
