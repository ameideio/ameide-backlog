# 621 — Agent Inner-Loop Test (Code)

This file is generated for review/traceability. It snapshots the current implementation sources that back:

- backlog/621-ameide-cli-inner-loop-test.md

> **Update (2026-01):**
>
> This snapshot is historical and includes legacy references (Tilt/Telepresence, “unit → integration → e2e” wording) that are no longer part of the normative 430v2 contract.
>
> Current contract:
> - `ameide dev inner-loop-test` runs Phase 0/1/2 only (local-only; no Kubernetes/Telepresence)
> - deployed-system E2E runs via `ameide ci e2e` (cluster-only; Playwright-only)
>
> Treat `backlog/430-unified-test-infrastructure-v2-target.md` as authoritative.

Generated at: 2026-01-08T16:16:01Z

## Tilt integration (Tiltfile excerpt)

```python
ci_settings(timeout='30m', readiness_timeout='15m')
TELEPRESENCE_BIN = os.getenv('TELEPRESENCE_BIN', 'telepresence')
TELEPRESENCE_ENV_ROOT = os.getenv('TELEPRESENCE_ENV_ROOT', '.telepresence-envs')
AMEIDE_BIN = os.getenv('AGENT_CI_AMEIDE_BIN', './ameide')
    },
    {
        'name': 'e2e-playwright-ci',
        'cmd': 'bash -lc "AGENT_CI_KUBE_CONTEXT={context} AGENT_CI_KUBE_NAMESPACE={namespace} AGENT_CI_TELEPRESENCE_TARGET={target} {ameide_bin} dev _inner-loop-test-phase3"'.format(
            context=_shell_quote(DEFAULT_CONTEXT),
            namespace=_shell_quote(DEFAULT_NAMESPACE),
            target=_shell_quote(TILT_TELEPRESENCE_TARGET),
            ameide_bin=_shell_quote(AMEIDE_BIN),
        ),
        'labels': ['tests', 'integration', 'e2e'],
        'resource_deps': [],
        # CI-only (run when explicitly enabled via `-- --resources e2e-playwright-ci`).
        'auto_init': True,
        'trigger_mode': TRIGGER_MODE_AUTO,
    },
```

## Go CLI entrypoints

## `packages/ameide_core_cli/internal/commands/dev.go`

```go
package commands

import (
	"fmt"
	"os"
	"path/filepath"

	// "github.com/fatih/color"
	"github.com/spf13/cobra"
)

// NewDevCommand creates the dev command
func NewDevCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "dev",
		Short: "Development environment commands",
		Long:  `Commands for managing the local development environment including Docker services.`,
	}

	cmd.AddCommand(newDevStartCommand())
	cmd.AddCommand(newDevStopCommand())
	cmd.AddCommand(newDevStatusCommand())
	cmd.AddCommand(newDevLogsCommand())
	cmd.AddCommand(newDevVerifyCommand())
	cmd.AddCommand(newDevInnerLoopTestCommand())
	cmd.AddCommand(newDevInnerLoopTestPhase3Command())
	cmd.AddCommand(newDevInnerLoopTestPhase3ConnectedCommand())
	cmd.AddCommand(newDevInnerLoopTestPhase3InnerCommand())

	return cmd
}

func newDevStartCommand() *cobra.Command {
	return &cobra.Command{
		Use:   "start",
		Short: "Start development environment",
		Long:  `Start all required services for local development using Docker Compose.`,
		RunE: func(cmd *cobra.Command, args []string) error {
			fmt.Println("Starting development environment...")

			// Bazel has been removed. Suggest using Tilt or docker-compose instead.
			fmt.Println("Bazel was removed from this repo.")
			fmt.Println("Use one of the following to start services:")
			fmt.Println("  • tilt up (recommended inside devcontainer)")
			fmt.Println("  • docker compose -f <your-compose> up -d")
			return nil

			fmt.Println("✓ Development environment started successfully")
			fmt.Println("\nServices:")
			fmt.Println("  • gRPC API: localhost:50051")
			fmt.Println("  • Envoy Proxy: localhost:8080")
			fmt.Println("  • Envoy Admin: localhost:9901")
			fmt.Println("  • PostgreSQL: localhost:5432")
			fmt.Println("  • Kafka: localhost:9092")
			fmt.Println("  • Prometheus: localhost:9090")
			fmt.Println("  • Grafana: localhost:3000")

			return nil
		},
	}
}

func newDevStopCommand() *cobra.Command {
	return &cobra.Command{
		Use:   "stop",
		Short: "Stop development environment",
		Long:  `Stop all running development services.`,
		RunE: func(cmd *cobra.Command, args []string) error {
			fmt.Println("Stopping development environment...")

			fmt.Println("Bazel was removed. Stop services with:")
			fmt.Println("  • tilt down   or   docker compose down")

			fmt.Println("✓ Development environment stopped")
			return nil
		},
	}
}

func newDevStatusCommand() *cobra.Command {
	return &cobra.Command{
		Use:     "status",
		Short:   "Show status of development services",
		Aliases: []string{"ps"},
		RunE: func(cmd *cobra.Command, args []string) error {
			fmt.Println("Bazel was removed. Check status with:")
			fmt.Println("  • tilt status   or   docker compose ps")
			return nil
		},
	}
}

func newDevLogsCommand() *cobra.Command {
	var follow bool
	var service string

	cmd := &cobra.Command{
		Use:   "logs [service]",
		Short: "View logs from development services",
		RunE: func(cmd *cobra.Command, args []string) error {
			fmt.Println("Bazel was removed. View logs with:")
			if follow {
				fmt.Println("  • tilt logs -f [svc]   or   docker compose logs -f [svc]")
			} else {
				fmt.Println("  • tilt logs [svc]      or   docker compose logs [svc]")
			}
			return nil
		},
	}

	cmd.Flags().BoolVarP(&follow, "follow", "f", false, "Follow log output")
	cmd.Flags().StringVarP(&service, "service", "s", "", "Service to show logs for")

	return cmd
}

// Helper to find graph root
func findRepoRoot() (string, error) {
	dir, err := os.Getwd()
	if err != nil {
		return "", err
	}

	for {
		// Detect repo root using common markers (no Bazel)
		markers := []string{"pnpm-workspace.yaml", ".git", "pyproject.toml", "package.json"}
		for _, m := range markers {
			if _, err := os.Stat(filepath.Join(dir, m)); err == nil {
				return dir, nil
			}
		}

		parent := filepath.Dir(dir)
		if parent == dir {
			break
		}
		dir = parent
	}

	return "", fmt.Errorf("could not find graph root (no repo markers found)")
}

```

## `packages/ameide_core_cli/internal/commands/dev_inner_loop.go`

```go
package commands

import (
	"context"
	"os"
	"os/signal"
	"syscall"

	"github.com/ameideio/ameide/packages/ameide_core_cli/internal/innerloop"
	"github.com/spf13/cobra"
)

func newDevInnerLoopTestCommand() *cobra.Command {
	return &cobra.Command{
		Use:   "inner-loop-test",
		Short: "Agent inner-loop verification (unit → integration → e2e)",
		Long:  "Runs the opinionated, fail-fast inner-loop verification sequence for coding agents: unit → integration → e2e (cluster via Tilt/Telepresence).",
		SilenceUsage: true,
		RunE: func(cmd *cobra.Command, args []string) error {
			baseCtx := cmd.Context()
			if baseCtx == nil {
				baseCtx = context.Background()
			}
			ctx, stop := signal.NotifyContext(baseCtx, os.Interrupt, syscall.SIGTERM)
			defer stop()

			repoRoot, err := innerloop.FindRepoRoot(ctx)
			if err != nil {
				return err
			}

			_, err = innerloop.Run(ctx, repoRoot)
			return err
		},
	}
}

```

## `packages/ameide_core_cli/internal/commands/dev_inner_loop_test_phase3.go`

```go
package commands

import (
	"context"
	"os"
	"os/signal"
	"syscall"

	"github.com/ameideio/ameide/packages/ameide_core_cli/internal/innerloop"
	"github.com/spf13/cobra"
)

func newDevInnerLoopTestPhase3Command() *cobra.Command {
	return &cobra.Command{
		Use:    "_inner-loop-test-phase3",
		Hidden: true,
		SilenceUsage: true,
		RunE: func(cmd *cobra.Command, args []string) error {
			baseCtx := cmd.Context()
			if baseCtx == nil {
				baseCtx = context.Background()
			}
			ctx, stop := signal.NotifyContext(baseCtx, os.Interrupt, syscall.SIGTERM)
			defer stop()
			return innerloop.RunPhase3TiltResourceFromEnv(ctx)
		},
	}
}

func newDevInnerLoopTestPhase3ConnectedCommand() *cobra.Command {
	return &cobra.Command{
		Use:    "_inner-loop-test-phase3-connected",
		Hidden: true,
		SilenceUsage: true,
		RunE: func(cmd *cobra.Command, args []string) error {
			baseCtx := cmd.Context()
			if baseCtx == nil {
				baseCtx = context.Background()
			}
			ctx, stop := signal.NotifyContext(baseCtx, os.Interrupt, syscall.SIGTERM)
			defer stop()
			return innerloop.RunPhase3ConnectedFromEnv(ctx)
		},
	}
}

func newDevInnerLoopTestPhase3InnerCommand() *cobra.Command {
	return &cobra.Command{
		Use:    "_inner-loop-test-phase3-inner",
		Hidden: true,
		SilenceUsage: true,
		RunE: func(cmd *cobra.Command, args []string) error {
			baseCtx := cmd.Context()
			if baseCtx == nil {
				baseCtx = context.Background()
			}
			ctx, stop := signal.NotifyContext(baseCtx, os.Interrupt, syscall.SIGTERM)
			defer stop()
			return innerloop.RunPhase3InnerFromEnv(ctx)
		},
	}
}

```

## Go implementation (modular runner)

## `packages/ameide_core_cli/internal/innerloop/innerloop.go`

```go
package innerloop

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"sort"
	"strings"
	"time"
)

type RunConfig struct {
	RepoRoot string
	RunRoot  string
}

type PhaseResult struct {
	Name     string
	Started  time.Time
	Finished time.Time
	Err      error
}

type RunResult struct {
	RunConfig
	Started  time.Time
	Finished time.Time
	Phases   []PhaseResult
}

func (r RunResult) SummaryLines() []string {
	lines := []string{
		fmt.Sprintf("[inner-loop-test] run_root=%s", r.RunRoot),
		fmt.Sprintf("[inner-loop-test] total=%s", r.Finished.Sub(r.Started).Round(time.Millisecond)),
	}
	for _, p := range r.Phases {
		d := p.Finished.Sub(p.Started).Round(time.Millisecond)
		status := "ok"
		if p.Err != nil {
			status = "fail"
		}
		lines = append(lines, fmt.Sprintf("[inner-loop-test] %s=%s (%s)", p.Name, d, status))
	}
	return lines
}

func FindRepoRoot(ctx context.Context) (string, error) {
	cmd := exec.CommandContext(ctx, "git", "rev-parse", "--show-toplevel")
	out, err := cmd.Output()
	if err == nil {
		root := strings.TrimSpace(string(out))
		if root != "" {
			return root, nil
		}
	}

	wd, err := os.Getwd()
	if err != nil {
		return "", err
	}
	dir := wd
	for {
		for _, marker := range []string{"pnpm-workspace.yaml", ".git", "pyproject.toml", "package.json", "go.work", "go.mod"} {
			if _, statErr := os.Stat(filepath.Join(dir, marker)); statErr == nil {
				return dir, nil
			}
		}
		parent := filepath.Dir(dir)
		if parent == dir {
			break
		}
		dir = parent
	}
	return "", fmt.Errorf("could not find repo root")
}

func EnsureDir0700(path string) error {
	return os.MkdirAll(path, 0o700)
}

func WriteFile0600(path string, data []byte) error {
	if err := EnsureDir0700(filepath.Dir(path)); err != nil {
		return err
	}
	return os.WriteFile(path, data, 0o600)
}

func Sha256FileHex(path string) (string, error) {
	f, err := os.Open(path)
	if err != nil {
		return "", err
	}
	defer f.Close()

	h := sha256.New()
	if _, err := io.Copy(h, f); err != nil {
		return "", err
	}
	return hex.EncodeToString(h.Sum(nil)), nil
}

func RelFromRepo(repoRoot, abs string) string {
	rel, err := filepath.Rel(repoRoot, abs)
	if err != nil {
		return abs
	}
	return filepath.ToSlash(rel)
}

func SanitizeForFilename(value string) string {
	value = strings.ReplaceAll(value, "/", "__")
	value = strings.ReplaceAll(value, "\\", "__")
	value = strings.ReplaceAll(value, " ", "_")
	value = strings.ReplaceAll(value, ":", "_")
	value = strings.ReplaceAll(value, "..", "__")
	return value
}

func SortStringsDedup(in []string) []string {
	if len(in) == 0 {
		return in
	}
	sort.Strings(in)
	out := make([]string, 0, len(in))
	last := ""
	for _, s := range in {
		if s == last {
			continue
		}
		out = append(out, s)
		last = s
	}
	return out
}

```

## `packages/ameide_core_cli/internal/innerloop/exec.go`

```go
package innerloop

import (
	"bufio"
	"context"
	"fmt"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"syscall"
	"time"
)

type ExecSpec struct {
	Dir     string
	Env     map[string]string
	LogFile string
	Command string
	Args    []string
}

func requireCmd(name string) error {
	_, err := exec.LookPath(name)
	if err != nil {
		return fmt.Errorf("missing required command %q (not in PATH)", name)
	}
	return nil
}

func mergeEnv(extra map[string]string) []string {
	if len(extra) == 0 {
		return os.Environ()
	}
	base := os.Environ()
	for k, v := range extra {
		base = append(base, fmt.Sprintf("%s=%s", k, v))
	}
	return base
}

func formatCmdLine(command string, args []string) string {
	parts := append([]string{command}, args...)
	return strings.Join(parts, " ")
}

func openLogFile(path string) (*os.File, error) {
	if path == "" {
		return nil, nil
	}
	if err := EnsureDir0700(filepath.Dir(path)); err != nil {
		return nil, err
	}
	return os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0o600)
}

func RunTee(ctx context.Context, spec ExecSpec) error {
	if err := requireCmd(spec.Command); err != nil {
		return err
	}

	logFile, err := openLogFile(spec.LogFile)
	if err != nil {
		return err
	}
	if logFile != nil {
		defer logFile.Close()
	}

	stdout := io.Writer(os.Stdout)
	stderr := io.Writer(os.Stderr)
	if logFile != nil {
		stdout = io.MultiWriter(os.Stdout, logFile)
		stderr = io.MultiWriter(os.Stderr, logFile)
	}

	line := fmt.Sprintf("[inner-loop-test] $ %s\n", formatCmdLine(spec.Command, spec.Args))
	if _, err := io.WriteString(stdout, line); err != nil {
		return err
	}

	cmd := exec.CommandContext(ctx, spec.Command, spec.Args...)
	cmd.Dir = spec.Dir
	cmd.Env = mergeEnv(spec.Env)
	cmd.Stdout = stdout
	cmd.Stderr = stderr
	return cmd.Run()
}

func RunTeeStdoutToFile(ctx context.Context, spec ExecSpec, stdoutFile string) error {
	if err := requireCmd(spec.Command); err != nil {
		return err
	}

	logFile, err := openLogFile(spec.LogFile)
	if err != nil {
		return err
	}
	if logFile != nil {
		defer logFile.Close()
	}
	var jsonFile *os.File
	if stdoutFile != "" {
		jsonFile, err = openLogFile(stdoutFile)
		if err != nil {
			return err
		}
		defer jsonFile.Close()
	}

	stdoutWriters := []io.Writer{os.Stdout}
	if logFile != nil {
		stdoutWriters = append(stdoutWriters, logFile)
	}
	if jsonFile != nil {
		stdoutWriters = append(stdoutWriters, jsonFile)
	}
	stdout := io.MultiWriter(stdoutWriters...)
	stderr := io.Writer(os.Stderr)
	if logFile != nil {
		stderr = io.MultiWriter(os.Stderr, logFile)
	}

	line := fmt.Sprintf("[inner-loop-test] $ %s\n", formatCmdLine(spec.Command, spec.Args))
	if _, err := io.WriteString(stdout, line); err != nil {
		return err
	}

	cmd := exec.CommandContext(ctx, spec.Command, spec.Args...)
	cmd.Dir = spec.Dir
	cmd.Env = mergeEnv(spec.Env)
	cmd.Stdout = stdout
	cmd.Stderr = stderr
	return cmd.Run()
}

// StartProcess starts a long-running process (e.g., Next dev) and tees output to the log file.
// The returned kill function attempts graceful termination, then SIGKILL.
func StartProcess(ctx context.Context, spec ExecSpec) (kill func() error, err error) {
	if err := requireCmd(spec.Command); err != nil {
		return nil, err
	}

	logFile, err := openLogFile(spec.LogFile)
	if err != nil {
		return nil, err
	}

	cmd := exec.CommandContext(ctx, spec.Command, spec.Args...)
	cmd.Dir = spec.Dir
	cmd.Env = mergeEnv(spec.Env)
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}

	stdoutPipe, err := cmd.StdoutPipe()
	if err != nil {
		logFile.Close()
		return nil, err
	}
	stderrPipe, err := cmd.StderrPipe()
	if err != nil {
		logFile.Close()
		return nil, err
	}

	if err := cmd.Start(); err != nil {
		logFile.Close()
		return nil, err
	}

	pgid := cmd.Process.Pid
	copyStream := func(r io.Reader) {
		defer func() { _ = r.(io.ReadCloser).Close() }()
		scanner := bufio.NewScanner(r)
		for scanner.Scan() {
			line := scanner.Text() + "\n"
			_, _ = io.WriteString(logFile, line)
		}
	}
	go copyStream(stdoutPipe)
	go copyStream(stderrPipe)

	killFn := func() error {
		defer logFile.Close()
		_ = syscall.Kill(-pgid, syscall.SIGTERM)
		time.Sleep(1 * time.Second)
		_ = syscall.Kill(-pgid, syscall.SIGKILL)
		return nil
	}

	go func() {
		<-ctx.Done()
		_ = killFn()
	}()

	return killFn, nil
}


```

## `packages/ameide_core_cli/internal/innerloop/git_drift.go`

```go
package innerloop

import (
	"context"
	"fmt"
	"os/exec"
	"path/filepath"
	"strings"
)

func GitStatusPorcelain(ctx context.Context, repoRoot string) (string, error) {
	cmd := exec.CommandContext(ctx, "git", "status", "--porcelain=v1")
	cmd.Dir = repoRoot
	out, err := cmd.Output()
	if err != nil {
		// Treat git status failure as "no status" (same spirit as the bash script).
		return "", nil
	}
	return string(out), nil
}

func WriteBaselineGitStatus(ctx context.Context, repoRoot, runRootAbs, runRootRel string) (baselinePath string, baseline string, err error) {
	if err := EnsureDir0700(runRootAbs); err != nil {
		return "", "", err
	}
	baseline, err = GitStatusPorcelain(ctx, repoRoot)
	if err != nil {
		return "", "", err
	}
	baselinePath = filepath.Join(runRootAbs, "git-status-baseline.txt")
	if err := WriteFile0600(baselinePath, []byte(baseline)); err != nil {
		return "", "", err
	}
	return baselinePath, baseline, nil
}

func FilterGitStatusOutsideRunRoot(status, runRootRel string) string {
	if runRootRel == "" {
		return status
	}
	prefix := runRootRel
	if !strings.HasSuffix(prefix, "/") {
		prefix += "/"
	}

	var b strings.Builder
	for _, line := range strings.Split(status, "\n") {
		if strings.TrimSpace(line) == "" {
			continue
		}
		if len(line) < 4 {
			continue
		}
		path := line[3:]
		if strings.HasPrefix(path, prefix) {
			continue
		}
		b.WriteString(line)
		b.WriteString("\n")
	}
	return b.String()
}

func AssertNoGitDrift(ctx context.Context, repoRoot, baseline, runRootRel string) error {
	current, err := GitStatusPorcelain(ctx, repoRoot)
	if err != nil {
		return err
	}
	currentFiltered := FilterGitStatusOutsideRunRoot(current, runRootRel)
	if currentFiltered != baseline {
		return fmt.Errorf("run introduced git drift outside RUN_ROOT (%s)", runRootRel)
	}
	return nil
}

```

## `packages/ameide_core_cli/internal/innerloop/run.go`

```go
package innerloop

import (
	"context"
	"fmt"
	"os"
	"path/filepath"
	"time"
)

func Run(ctx context.Context, repoRoot string) (RunResult, error) {
	started := time.Now()

	ts := time.Now().UTC().Format("20060102T150405Z")
	runRoot := filepath.Join(repoRoot, "artifacts", "agent-ci", ts)
	cfg := RunConfig{RepoRoot: repoRoot, RunRoot: runRoot}
	result := RunResult{RunConfig: cfg, Started: started}

	if err := EnsureDir0700(filepath.Join(runRoot, "logs")); err != nil {
		result.Finished = time.Now()
		return result, err
	}

	runRootRel := RelFromRepo(repoRoot, runRoot)
	_, baseline, err := WriteBaselineGitStatus(ctx, repoRoot, runRoot, runRootRel)
	if err != nil {
		result.Finished = time.Now()
		return result, err
	}

	defer func() {
		result.Finished = time.Now()
		for _, line := range result.SummaryLines() {
			fmt.Fprintln(os.Stdout, line)
		}
	}()

	phases := []struct {
		name string
		fn   func(context.Context, RunConfig) error
	}{
		{name: "phase1", fn: RunPhase1},
		{name: "phase2", fn: RunPhase2},
		{name: "phase3", fn: RunPhase3},
	}

	for _, p := range phases {
		pr := PhaseResult{Name: p.name, Started: time.Now()}
		err := p.fn(ctx, cfg)
		pr.Finished = time.Now()
		pr.Err = err
		result.Phases = append(result.Phases, pr)
		if err != nil {
			return result, err
		}
	}

	if err := AssertNoGitDrift(ctx, repoRoot, baseline, runRootRel); err != nil {
		return result, err
	}

	return result, nil
}


```

## `packages/ameide_core_cli/internal/innerloop/phase1.go`

```go
package innerloop

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
)

func RunPhase1(ctx context.Context, cfg RunConfig) error {
	logRoot := filepath.Join(cfg.RunRoot, "logs", "phase1")
	if err := EnsureDir0700(logRoot); err != nil {
		return err
	}
	if err := EnsureDir0700(filepath.Join(cfg.RunRoot, "test-results")); err != nil {
		return err
	}

	if err := runPhase1Node(ctx, cfg, logRoot); err != nil {
		return err
	}
	if err := runPhase1Python(ctx, cfg, logRoot); err != nil {
		return err
	}
	if err := runPhase1Go(ctx, cfg, logRoot); err != nil {
		return err
	}
	return nil
}

func runPhase1Node(ctx context.Context, cfg RunConfig, logRoot string) error {
	if !fileExists(filepath.Join(cfg.RepoRoot, "package.json")) || !fileExists(filepath.Join(cfg.RepoRoot, "pnpm-lock.yaml")) {
		return nil
	}
	if err := requireCmd("pnpm"); err != nil {
		return err
	}

	nodeModulesOK := true
	modulesYaml := filepath.Join(cfg.RepoRoot, "node_modules", ".modules.yaml")
	if !dirExists(filepath.Join(cfg.RepoRoot, "node_modules")) || !fileExists(modulesYaml) {
		nodeModulesOK = false
	} else if !fileExists(filepath.Join(cfg.RepoRoot, "node_modules", ".bin", "eslint")) || !fileExists(filepath.Join(cfg.RepoRoot, "node_modules", ".bin", "tsc")) {
		nodeModulesOK = false
	} else {
		lockInfo, err1 := os.Stat(filepath.Join(cfg.RepoRoot, "pnpm-lock.yaml"))
		modInfo, err2 := os.Stat(modulesYaml)
		if err1 != nil || err2 != nil {
			nodeModulesOK = false
		} else if lockInfo.ModTime().After(modInfo.ModTime()) {
			nodeModulesOK = false
		}
	}

	if !nodeModulesOK {
		if err := RunTee(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			LogFile: filepath.Join(logRoot, "pnpm-install.log"),
			Command: "pnpm",
			Args:    []string{"install", "--frozen-lockfile"},
		}); err != nil {
			return err
		}
	}

	_ = os.RemoveAll(filepath.Join(cfg.RepoRoot, "services", "www_ameide_platform", ".next", "dev", "types"))

	for _, step := range []struct {
		name string
		args []string
	}{
		{name: "pnpm-lint.log", args: []string{"lint"}},
		{name: "pnpm-typecheck.log", args: []string{"typecheck"}},
		{name: "pnpm-test-unit.log", args: []string{"test:unit"}},
	} {
		if err := RunTee(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			LogFile: filepath.Join(logRoot, step.name),
			Command: "pnpm",
			Args:    step.args,
		}); err != nil {
			return err
		}
	}

	return nil
}

func runPhase1Python(ctx context.Context, cfg RunConfig, logRoot string) error {
	if !fileExists(filepath.Join(cfg.RepoRoot, "uv.lock")) || !fileExists(filepath.Join(cfg.RepoRoot, "pyproject.toml")) {
		return nil
	}
	if err := requireCmd("uv"); err != nil {
		return err
	}

	venvOK := fileExists(filepath.Join(cfg.RepoRoot, ".venv", "bin", "python"))
	uvStamp := filepath.Join(cfg.RepoRoot, ".venv", ".agent-ci-uv-lock.sha256")
	if venvOK {
		uvHash, err := Sha256FileHex(filepath.Join(cfg.RepoRoot, "uv.lock"))
		if err != nil {
			venvOK = false
		} else {
			stamp, _ := os.ReadFile(uvStamp)
			if strings.TrimSpace(string(stamp)) != uvHash {
				venvOK = false
			}
		}
	}

	if !venvOK {
		if err := RunTee(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			LogFile: filepath.Join(logRoot, "uv-sync.log"),
			Command: "uv",
			Args:    []string{"sync", "--all-packages", "--dev"},
		}); err != nil {
			return err
		}
		uvHash, err := Sha256FileHex(filepath.Join(cfg.RepoRoot, "uv.lock"))
		if err != nil {
			return err
		}
		if err := EnsureDir0700(filepath.Dir(uvStamp)); err != nil {
			return err
		}
		if err := os.WriteFile(uvStamp, []byte(uvHash+"\n"), 0o600); err != nil {
			return err
		}
	}

	ruffTargets := []string{
		"services/inference",
		"packages/ameide_core_domain",
		"packages/ameide_core_inference_agents",
		"packages/ameide_core_platform_common",
	}
	ruffExisting := make([]string, 0, len(ruffTargets))
	for _, p := range ruffTargets {
		if pathExists(filepath.Join(cfg.RepoRoot, p)) {
			ruffExisting = append(ruffExisting, p)
		}
	}
	if len(ruffExisting) > 0 {
		if err := requireCmd("uvx"); err != nil {
			return err
		}
		args := append([]string{"ruff", "check"}, ruffExisting...)
		if err := RunTee(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			LogFile: filepath.Join(logRoot, "ruff-check.log"),
			Command: "uvx",
			Args:    args,
		}); err != nil {
			return err
		}
	}

	if dirExists(filepath.Join(cfg.RepoRoot, "packages")) {
		junit := filepath.Join(cfg.RunRoot, "test-results", "pytest-packages.xml")
		if err := RunTee(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			LogFile: filepath.Join(logRoot, "pytest-packages.log"),
			Command: "uv",
			Args:    []string{"run", "-m", "pytest", "-q", "packages", "--ignore-glob=*/__tests__/integration/*", "--junitxml=" + junit},
		}); err != nil {
			return err
		}
	}
	return nil
}

func runPhase1Go(ctx context.Context, cfg RunConfig, logRoot string) error {
	if !fileExists(filepath.Join(cfg.RepoRoot, "go.work")) && !fileExists(filepath.Join(cfg.RepoRoot, "go.mod")) {
		return nil
	}
	if err := requireCmd("go"); err != nil {
		return err
	}

	pkgs, err := goListPackages(ctx, cfg.RepoRoot)
	if err != nil {
		return err
	}
	filtered := make([]string, 0, len(pkgs))
	for _, p := range pkgs {
		if strings.Contains(p, "/__tests__/integration") || strings.Contains(p, "/tests/integration") {
			continue
		}
		filtered = append(filtered, p)
	}
	if len(filtered) == 0 {
		return nil
	}

	args := append([]string{"test"}, filtered...)
	return RunTee(ctx, ExecSpec{
		Dir:     cfg.RepoRoot,
		LogFile: filepath.Join(logRoot, "go-test-unit.log"),
		Command: "go",
		Args:    args,
	})
}

func goListPackages(ctx context.Context, repoRoot string) ([]string, error) {
	cmd := exec.CommandContext(ctx, "go", "list", "./...")
	cmd.Dir = repoRoot
	out, err := cmd.Output()
	if err != nil {
		return nil, fmt.Errorf("go list failed: %w", err)
	}
	lines := strings.Split(string(out), "\n")
	pkgs := make([]string, 0, len(lines))
	for _, l := range lines {
		l = strings.TrimSpace(l)
		if l == "" {
			continue
		}
		pkgs = append(pkgs, l)
	}
	return pkgs, nil
}

func pathExists(path string) bool {
	_, err := os.Stat(path)
	return err == nil
}

func fileExists(path string) bool {
	info, err := os.Stat(path)
	return err == nil && info.Mode().IsRegular()
}

func dirExists(path string) bool {
	info, err := os.Stat(path)
	return err == nil && info.IsDir()
}

```

## `packages/ameide_core_cli/internal/innerloop/phase2.go`

```go
package innerloop

import (
	"context"
	"encoding/json"
	"fmt"
	"io/fs"
	"os"
	"os/exec"
	"path/filepath"
	"sort"
	"strings"
)

func RunPhase2(ctx context.Context, cfg RunConfig) error {
	logRoot := filepath.Join(cfg.RunRoot, "logs", "phase2")
	if err := EnsureDir0700(logRoot); err != nil {
		return err
	}
	resultsRoot := filepath.Join(cfg.RunRoot, "test-results", "phase2")
	for _, p := range []string{
		filepath.Join(resultsRoot, "go"),
		filepath.Join(resultsRoot, "jest"),
		filepath.Join(resultsRoot, "pytest"),
	} {
		if err := EnsureDir0700(p); err != nil {
			return err
		}
	}

	integrationDirs, err := DiscoverIntegrationDirs(cfg.RepoRoot)
	if err != nil {
		return err
	}
	integrationDirs = SortStringsDedup(integrationDirs)
	if len(integrationDirs) == 0 {
		return nil
	}

	var goPkgs []string
	jestWorkspaceSet := map[string]struct{}{}
	var jestWorkspaces []string
	type pyEntry struct {
		rel    string
		log    string
		junit  string
	}
	var pyEntries []pyEntry

	for _, absDir := range integrationDirs {
		rel := RelFromRepo(cfg.RepoRoot, absDir)
		logFile := filepath.Join(logRoot, SanitizeForFilename(rel)+".log")
		_ = WriteFile0600(logFile, []byte(fmt.Sprintf("[inner-loop-test] === integration: %s ===\n", rel)))

		ranAny := false
		if HasGoTests(absDir) {
			ranAny = true
			goPkgs = append(goPkgs, "./"+rel+"/...")
			appendLine(logFile, "[inner-loop-test] (note) go tests are batched; see go__integration.log\n")
		}

		if HasPyTests(absDir) {
			ranAny = true
			junit := filepath.Join(resultsRoot, "pytest", SanitizeForFilename(rel)+".xml")
			pyEntries = append(pyEntries, pyEntry{rel: rel, log: logFile, junit: junit})
		}

		if HasJestTests(absDir) {
			ranAny = true
			pkgDir := FindOwningPackageDir(cfg.RepoRoot, absDir)
			if pkgDir != "" {
				if _, ok := jestWorkspaceSet[pkgDir]; !ok {
					jestWorkspaceSet[pkgDir] = struct{}{}
					jestWorkspaces = append(jestWorkspaces, pkgDir)
				}
				wsRel := RelFromRepo(cfg.RepoRoot, pkgDir)
				wsLog := "node__" + SanitizeForFilename(wsRel) + ".log"
				appendLine(logFile, fmt.Sprintf("[inner-loop-test] (note) jest is batched per workspace; see %s\n", wsLog))
			}
		}

		if !ranAny {
			appendLine(logFile, "[inner-loop-test] (skip) no test files detected\n")
		}
	}

	envBase := map[string]string{"INTEGRATION_MODE": "repo"}

	if len(goPkgs) > 0 {
		if err := requireCmd("go"); err != nil {
			return err
		}
		goLog := filepath.Join(logRoot, "go__integration.log")
		goJSON := filepath.Join(resultsRoot, "go", "go__integration.json")
		args := append([]string{"test", "-json"}, goPkgs...)
		if err := RunTeeStdoutToFile(ctx, ExecSpec{
			Dir:     cfg.RepoRoot,
			Env:     envBase,
			LogFile: goLog,
			Command: "go",
			Args:    args,
		}, goJSON); err != nil {
			return err
		}
	}

	if len(pyEntries) > 0 {
		if err := requireCmd("uv"); err != nil {
			return err
		}
		pyPath := filepath.Join(cfg.RepoRoot, "tools", "integration-runner")
		if existing := os.Getenv("PYTHONPATH"); strings.TrimSpace(existing) != "" {
			pyPath = pyPath + string(os.PathListSeparator) + existing
		}
		for _, e := range pyEntries {
			env := map[string]string{
				"INTEGRATION_MODE": "repo",
				"PYTHONPATH":       pyPath,
			}
			args := []string{"run", "-m", "pytest", "-q", "-x", "--junitxml=" + e.junit, e.rel}
			if err := RunTee(ctx, ExecSpec{
				Dir:     cfg.RepoRoot,
				Env:     env,
				LogFile: e.log,
				Command: "uv",
				Args:    args,
			}); err != nil {
				return err
			}
		}
	}

	if len(jestWorkspaces) > 0 {
		if err := requireCmd("pnpm"); err != nil {
			return err
		}

		sort.Strings(jestWorkspaces)
		jestFlagCache := map[string]string{}
		for _, pkgDir := range jestWorkspaces {
			wsRel := RelFromRepo(cfg.RepoRoot, pkgDir)
			wsLog := filepath.Join(logRoot, "node__"+SanitizeForFilename(wsRel)+".log")

			jestConfigArgs := []string{}
			if fileExists(filepath.Join(pkgDir, "jest.integration.config.js")) {
				jestConfigArgs = []string{"--config", "jest.integration.config.js"}
			}

			flag := jestFlagCache[pkgDir]
			if flag == "" {
				detected, err := detectJestTestPathFlag(ctx, pkgDir)
				if err != nil {
					return err
				}
				flag = detected
				jestFlagCache[pkgDir] = detected
			}

			testPathArgs := []string{}
			if flag != "" {
				testPathArgs = []string{flag, "(__tests__/integration|tests/integration)"}
			}

			junitName := "jest-" + SanitizeForFilename(wsRel) + ".xml"
			env := map[string]string{
				"INTEGRATION_MODE":       "repo",
				"JEST_JUNIT_OUTPUT_DIR":  filepath.ToSlash(filepath.Join(resultsRoot, "jest")),
				"JEST_JUNIT_OUTPUT_NAME": junitName,
			}

			args := []string{"exec", "jest"}
			args = append(args, jestConfigArgs...)
			args = append(args,
				"--bail=1",
				"--passWithNoTests",
				"--maxWorkers=50%",
				"--reporters=default",
				"--reporters=jest-junit",
			)
			args = append(args, testPathArgs...)

			if err := RunTee(ctx, ExecSpec{
				Dir:     pkgDir,
				Env:     env,
				LogFile: wsLog,
				Command: "pnpm",
				Args:    args,
			}); err != nil {
				return err
			}
		}
	}

	return nil
}

func appendLine(path, line string) {
	f, err := os.OpenFile(path, os.O_WRONLY|os.O_APPEND, 0o600)
	if err != nil {
		return
	}
	defer f.Close()
	_, _ = f.WriteString(line)
}

func DiscoverIntegrationDirs(repoRoot string) ([]string, error) {
	roots := []string{"services", "packages", "primitives", "capabilities", "tests"}
	var searchRoots []string
	for _, r := range roots {
		p := filepath.Join(repoRoot, r)
		if dirExists(p) {
			searchRoots = append(searchRoots, p)
		}
	}
	if len(searchRoots) == 0 {
		return nil, nil
	}

	pruneNames := map[string]struct{}{
		"node_modules":   {},
		".venv":          {},
		"venv":           {},
		"__pycache__":    {},
		".pytest_cache":  {},
		".ruff_cache":    {},
		".mypy_cache":    {},
		".cache":         {},
		"~":              {},
		"dist":           {},
		"build":          {},
		"out":            {},
		"tmp":            {},
		"artifacts":      {},
		".next":          {},
	}

	var found []string
	for _, root := range searchRoots {
		err := filepath.WalkDir(root, func(path string, d os.DirEntry, err error) error {
			if err != nil {
				return err
			}
			if !d.IsDir() {
				return nil
			}
			if _, ok := pruneNames[d.Name()]; ok {
				return filepath.SkipDir
			}
			slash := filepath.ToSlash(path)
			if strings.HasSuffix(slash, "/__tests__/integration") || strings.HasSuffix(slash, "/tests/integration") {
				found = append(found, path)
				return filepath.SkipDir
			}
			return nil
		})
		if err != nil {
			return nil, err
		}
	}
	return found, nil
}

func FindOwningPackageDir(repoRoot, startDir string) string {
	dir := startDir
	for dir != repoRoot && dir != string(filepath.Separator) {
		if fileExists(filepath.Join(dir, "package.json")) {
			return dir
		}
		parent := filepath.Dir(dir)
		if parent == dir {
			break
		}
		dir = parent
	}
	if fileExists(filepath.Join(repoRoot, "package.json")) {
		return repoRoot
	}
	return ""
}

func HasGoTests(dir string) bool {
	return hasFileMatch(dir, func(name string) bool { return strings.HasSuffix(name, "_test.go") })
}

func HasPyTests(dir string) bool {
	return hasFileMatch(dir, func(name string) bool {
		return strings.HasPrefix(name, "test_") && strings.HasSuffix(name, ".py") || strings.HasSuffix(name, "_test.py")
	})
}

func HasJestTests(dir string) bool {
	return hasFileMatch(dir, func(name string) bool {
		if strings.HasSuffix(name, ".test.ts") || strings.HasSuffix(name, ".spec.ts") || strings.HasSuffix(name, ".test.tsx") || strings.HasSuffix(name, ".spec.tsx") {
			return true
		}
		if strings.HasSuffix(name, ".test.js") || strings.HasSuffix(name, ".spec.js") || strings.HasSuffix(name, ".test.jsx") || strings.HasSuffix(name, ".spec.jsx") {
			return true
		}
		return false
	})
}

func hasFileMatch(root string, match func(name string) bool) bool {
	found := false
	_ = filepath.WalkDir(root, func(path string, d os.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() {
			return nil
		}
		if match(d.Name()) {
			found = true
			return fs.SkipAll
		}
		return nil
	})
	return found
}

func detectJestTestPathFlag(ctx context.Context, pkgDir string) (string, error) {
	// Prefer explicit capability detection so we work with Jest 29 and 30.
	cmd := exec.CommandContext(ctx, "pnpm", "exec", "jest", "--help")
	cmd.Dir = pkgDir
	out, _ := cmd.CombinedOutput()
	s := string(out)
	if strings.Contains(s, "--testPathPatterns") {
		return "--testPathPatterns", nil
	}
	if strings.Contains(s, "--testPathPattern") {
		return "--testPathPattern", nil
	}
	return "", nil
}

// Debug helper: keep JSON parsing for Service/envoy-grpc in one place.
type k8sService struct {
	Spec struct {
		Type         string `json:"type"`
		ExternalName string `json:"externalName"`
		Ports        []struct {
			Port int `json:"port"`
		} `json:"ports"`
	} `json:"spec"`
}

func parseK8sServiceJSON(data []byte) (k8sService, error) {
	var svc k8sService
	if err := json.Unmarshal(data, &svc); err != nil {
		return k8sService{}, err
	}
	return svc, nil
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3.go`

```go
package innerloop

import (
	"context"
	"path/filepath"
)

func RunPhase3(ctx context.Context, cfg RunConfig) error {
	logRoot := filepath.Join(cfg.RunRoot, "logs", "phase3")
	if err := EnsureDir0700(logRoot); err != nil {
		return err
	}

	if err := requireCmd("tilt"); err != nil {
		return err
	}
	if err := requireCmd("go"); err != nil {
		return err
	}

	ameideBin := filepath.Join(cfg.RunRoot, "bin", "ameide")
	if err := EnsureDir0700(filepath.Dir(ameideBin)); err != nil {
		return err
	}
	if err := RunTee(ctx, ExecSpec{
		Dir:     cfg.RepoRoot,
		LogFile: filepath.Join(logRoot, "go-build-ameide.log"),
		Command: "go",
		Args:    []string{"build", "-o", ameideBin, "./packages/ameide_core_cli/cmd/ameide"},
	}); err != nil {
		return err
	}

	env := map[string]string{
		"AGENT_CI_RUN_ROOT":  cfg.RunRoot,
		"AGENT_CI_AMEIDE_BIN": ameideBin,
	}

	tiltSnapshot := filepath.Join(logRoot, "tilt-snapshot.json")
	return RunTee(ctx, ExecSpec{
		Dir:     cfg.RepoRoot,
		Env:     env,
		LogFile: filepath.Join(logRoot, "tilt-ci.log"),
		Command: "tilt",
		Args:    []string{"ci", "--timeout", "30m", "--output-snapshot-on-exit", tiltSnapshot, "--", "--resources", "e2e-playwright-ci"},
	})
}


```

## `packages/ameide_core_cli/internal/innerloop/phase3_env.go`

```go
package innerloop

import (
	"os"
	"path/filepath"
	"regexp"
	"strings"
	"time"
)

type phase3Env struct {
	RepoRoot           string
	RunRoot            string
	TelepresenceBin    string
	KubeContext        string
	KubeNamespace      string
	TelepresenceTarget string
	BaseURL            string

	SecretNamespace string
	SecretName      string

	AgentID       string
	InterceptName string
	Workload      string
	EnvJSONFile   string
	GrpcBaseURL   string
}

func loadPhase3Env(repoRoot string) (phase3Env, error) {
	telepresenceBin := os.Getenv("TELEPRESENCE_BIN")
	if telepresenceBin == "" {
		telepresenceBin = "telepresence"
	}

	target := os.Getenv("AGENT_CI_TELEPRESENCE_TARGET")
	if target == "" {
		target = "ameide-aks"
	}

	defaultBaseURL := "https://platform.dev.ameide.io"
	if target == "ameide-local" {
		defaultBaseURL = "https://platform.local.ameide.io"
	}
	baseURL := firstNonEmpty(os.Getenv("BASE_URL"), os.Getenv("WWW_AMEIDE_PLATFORM_BASE_URL"), defaultBaseURL)

	kubeContext := firstNonEmpty(os.Getenv("AGENT_CI_KUBE_CONTEXT"), os.Getenv("TELEPRESENCE_CONTEXT"), os.Getenv("TILT_CONTEXT"))
	kubeNamespace := firstNonEmpty(os.Getenv("AGENT_CI_KUBE_NAMESPACE"), os.Getenv("TELEPRESENCE_NAMESPACE"), os.Getenv("TILT_NAMESPACE"), "ameide-dev")

	secretNamespace := firstNonEmpty(os.Getenv("WWW_AMEIDE_PLATFORM_E2E_NAMESPACE"), kubeNamespace)
	secretName := firstNonEmpty(os.Getenv("WWW_AMEIDE_PLATFORM_E2E_SECRET_NAME"), "playwright-int-tests-secrets")

	runRoot := os.Getenv("AGENT_CI_RUN_ROOT")
	if runRoot == "" {
		ts := time.Now().UTC().Format("20060102T150405Z")
		runRoot = filepath.Join(repoRoot, "artifacts", "agent-ci", "tilt-e2e-"+ts)
	}

	agentRaw := firstNonEmpty(os.Getenv("AMEIDE_AGENT_ID"), os.Getenv("HOSTNAME"), "agent")
	agentID := sanitizeAgentID(agentRaw)
	_ = os.Setenv("AMEIDE_AGENT_ID", agentID)

	interceptName := "agent-ci-www-platform-" + agentID
	workload := "www-ameide-platform"

	tmpDir := "/dev/shm"
	if !dirExists(tmpDir) {
		tmpDir = os.TempDir()
	}
	envJSONFile := filepath.Join(tmpDir, "telepresence-"+interceptName+"-env.json")

	return phase3Env{
		RepoRoot:           repoRoot,
		RunRoot:            runRoot,
		TelepresenceBin:    telepresenceBin,
		KubeContext:        kubeContext,
		KubeNamespace:      kubeNamespace,
		TelepresenceTarget: target,
		BaseURL:            baseURL,
		SecretNamespace:    secretNamespace,
		SecretName:         secretName,
		AgentID:            agentID,
		InterceptName:      interceptName,
		Workload:           workload,
		EnvJSONFile:        envJSONFile,
	}, nil
}

func firstNonEmpty(values ...string) string {
	for _, v := range values {
		if strings.TrimSpace(v) != "" {
			return v
		}
	}
	return ""
}

func sanitizeAgentID(raw string) string {
	raw = strings.ToLower(raw)
	raw = regexp.MustCompile(`[^a-z0-9-]+`).ReplaceAllString(raw, "-")
	raw = strings.Trim(raw, "-")
	if len(raw) > 48 {
		raw = raw[:48]
	}
	if raw == "" {
		return "agent"
	}
	return raw
}


```

## `packages/ameide_core_cli/internal/innerloop/phase3_telepresence.go`

```go
package innerloop

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os/exec"
	"path/filepath"
	"strconv"
	"strings"
	"time"
)

type telepresenceClient struct {
	bin string
}

func newTelepresenceClient(bin string) *telepresenceClient {
	return &telepresenceClient{bin: bin}
}

func (t *telepresenceClient) QuitStopDaemons(ctx context.Context) {
	_ = exec.CommandContext(ctx, t.bin, "quit", "-s").Run()
}

func (t *telepresenceClient) Leave(ctx context.Context, interceptName string) {
	_ = exec.CommandContext(ctx, t.bin, "leave", interceptName).Run()
}

func (t *telepresenceClient) SupportsHTTPHeaderIntercept(ctx context.Context) (bool, error) {
	out, err := exec.CommandContext(ctx, t.bin, "intercept", "--help").CombinedOutput()
	if err != nil {
		return false, err
	}
	return strings.Contains(string(out), "--http-header"), nil
}

func (t *telepresenceClient) GatherLogsZip(ctx context.Context, runRoot string) {
	out := filepath.Join(runRoot, "logs", "phase3", "telepresence_logs.zip")
	_ = EnsureDir0700(filepath.Dir(out))
	c, cancel := context.WithTimeout(ctx, 90*time.Second)
	defer cancel()
	_ = exec.CommandContext(c, t.bin, "gather-logs", "-o", out, "--get-pod-yaml").Run()
}

type tpStatus struct {
	Connections []tpConnection `json:"connections"`
}

type tpConnection struct {
	UserDaemon struct {
		Running bool   `json:"running"`
		Status  string `json:"status"`
	} `json:"user_daemon"`
	RootDaemon struct {
		Running bool   `json:"running"`
		Version string `json:"version"`
	} `json:"root_daemon"`
	TrafficManager struct {
		Version string `json:"version"`
	} `json:"traffic_manager"`
}

func (t *telepresenceClient) StatusJSON(ctx context.Context) ([]byte, tpStatus, error) {
	out, err := exec.CommandContext(ctx, t.bin, "status", "--output", "json", "--multi-daemon").Output()
	if err != nil {
		return nil, tpStatus{}, err
	}
	var status tpStatus
	if err := json.Unmarshal(out, &status); err != nil {
		return out, tpStatus{}, err
	}
	return out, status, nil
}

func (t *telepresenceClient) TrafficManagerVersion(ctx context.Context) (string, error) {
	_, status, err := t.StatusJSON(ctx)
	if err != nil {
		return "", fmt.Errorf("unable to determine Telepresence Traffic Manager version from `telepresence status`")
	}
	for _, c := range status.Connections {
		if strings.TrimSpace(c.TrafficManager.Version) != "" {
			return strings.TrimSpace(c.TrafficManager.Version), nil
		}
	}
	return "", fmt.Errorf("unable to determine Telepresence Traffic Manager version from `telepresence status`")
}

func (t *telepresenceClient) WriteStatusArtifact(ctx context.Context, runRoot string) {
	raw, _, err := t.StatusJSON(ctx)
	if err != nil || len(raw) == 0 {
		return
	}
	path := filepath.Join(runRoot, "logs", "phase3", "telepresence-status.json")
	_ = WriteFile0600(path, raw)
}

func semverGE(a, b string) bool {
	parse := func(v string) (int, int, int) {
		v = strings.TrimPrefix(v, "v")
		parts := strings.Split(v, ".")
		toInt := func(i int) int {
			if i >= len(parts) {
				return 0
			}
			n, _ := strconv.Atoi(parts[i])
			return n
		}
		return toInt(0), toInt(1), toInt(2)
	}
	a1, a2, a3 := parse(a)
	b1, b2, b3 := parse(b)
	if a1 != b1 {
		return a1 > b1
	}
	if a2 != b2 {
		return a2 > b2
	}
	return a3 >= b3
}

func contextExitCode(err error) int {
	if err == nil {
		return 0
	}
	if errors.Is(err, context.DeadlineExceeded) {
		return 124
	}
	if errors.Is(err, context.Canceled) {
		return 137
	}
	return 1
}

func ensureNoTelepresenceStaleSession(ctx context.Context, tp *telepresenceClient) {
	// This tool must leave the workstation clean. Also helps avoid "Cluster configuration changed" errors.
	tp.QuitStopDaemons(ctx)
	// If Telepresence reports stale daemons, we still keep going; connect will error if it can't recover.
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3_kube.go`

```go
package innerloop

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

type kubeClient struct {
	clientset *kubernetes.Clientset
}

func newKubeClient(ctx context.Context, kubeContext string) (*kubeClient, error) {
	loadingRules := clientcmd.NewDefaultClientConfigLoadingRules()
	overrides := &clientcmd.ConfigOverrides{}
	if kubeContext != "" {
		overrides.CurrentContext = kubeContext
	}
	restCfg, err := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(loadingRules, overrides).ClientConfig()
	if err != nil {
		return nil, err
	}
	clientset, err := kubernetes.NewForConfig(restCfg)
	if err != nil {
		return nil, err
	}
	return &kubeClient{clientset: clientset}, nil
}

func (k *kubeClient) ResolveEnvoyGrpcBaseURL(ctx context.Context, namespace string) (string, error) {
	svc, err := k.clientset.CoreV1().Services(namespace).Get(ctx, "envoy-grpc", metav1.GetOptions{})
	if err != nil {
		if apierrors.IsNotFound(err) {
			return "", fmt.Errorf("missing Service/envoy-grpc in namespace %s (required for local SSR to reach the platform gateway)", namespace)
		}
		return "", err
	}

	port := int32(9000)
	if len(svc.Spec.Ports) > 0 && svc.Spec.Ports[0].Port != 0 {
		port = svc.Spec.Ports[0].Port
	}

	host := fmt.Sprintf("envoy-grpc.%s.svc.cluster.local", namespace)
	if svc.Spec.Type == corev1.ServiceTypeExternalName && svc.Spec.ExternalName != "" {
		host = svc.Spec.ExternalName
	}

	return fmt.Sprintf("http://%s:%d", host, port), nil
}

func (k *kubeClient) EnsureWorkloadInjectionEnabled(ctx context.Context, env phase3Env) error {
	dep, err := k.clientset.AppsV1().Deployments(env.KubeNamespace).Get(ctx, env.Workload, metav1.GetOptions{})
	if err != nil {
		return err
	}

	ann := dep.Spec.Template.Annotations
	if ann != nil {
		if v := ann["telepresence.io/inject-traffic-agent"]; v != "" {
			return nil
		}
	}

	if env.TelepresenceTarget != "ameide-local" {
		return fmt.Errorf("deployment/%s is missing annotation telepresence.io/inject-traffic-agent; Telepresence intercepts can hang when manager uses WhenEnabled policy", env.Workload)
	}

	patch := map[string]any{
		"spec": map[string]any{
			"template": map[string]any{
				"metadata": map[string]any{
					"annotations": map[string]string{
						"telepresence.io/inject-traffic-agent": "enabled",
					},
				},
			},
		},
	}
	b, _ := json.Marshal(patch)
	if _, err := k.clientset.AppsV1().Deployments(env.KubeNamespace).Patch(ctx, env.Workload, types.MergePatchType, b, metav1.PatchOptions{}); err != nil {
		return err
	}
	return waitDeploymentRollout(ctx, k.clientset, env.KubeNamespace, env.Workload, 240*time.Second)
}

func waitDeploymentRollout(ctx context.Context, clientset *kubernetes.Clientset, namespace, name string, timeout time.Duration) error {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	for {
		select {
		case <-ctx.Done():
			return fmt.Errorf("deployment/%s rollout timed out", name)
		case <-time.After(1 * time.Second):
		}

		dep, err := clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
		if err != nil {
			return err
		}
		if deploymentRolledOut(dep) {
			return nil
		}
	}
}

func deploymentRolledOut(dep *appsv1.Deployment) bool {
	desired := int32(1)
	if dep.Spec.Replicas != nil {
		desired = *dep.Spec.Replicas
	}
	if dep.Generation > dep.Status.ObservedGeneration {
		return false
	}
	if dep.Status.UpdatedReplicas < desired {
		return false
	}
	if dep.Status.AvailableReplicas < desired {
		return false
	}
	return true
}

func (k *kubeClient) GetSecret(ctx context.Context, namespace, name string) (*corev1.Secret, error) {
	secret, err := k.clientset.CoreV1().Secrets(namespace).Get(ctx, name, metav1.GetOptions{})
	if err != nil {
		return nil, err
	}
	return secret, nil
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3_secrets.go`

```go
package innerloop

import (
	"context"
	"fmt"
	"os"
)

func loadPlaywrightSecrets(ctx context.Context, kc *kubeClient, env phase3Env) error {
	_ = os.Setenv("E2E_SSO_USERNAME", "owner@acme.test")
	_ = os.Setenv("E2E_OWNER_USERNAME", "owner@acme.test")
	_ = os.Setenv("E2E_VIEWER_USERNAME", "viewer@acme.test")
	_ = os.Setenv("E2E_NEWMEMBER_USERNAME", "newmember@example.com")

	secret, err := kc.GetSecret(ctx, env.SecretNamespace, env.SecretName)
	if err != nil {
		return err
	}

	require := func(key string) (string, error) {
		value := secret.Data[key]
		if len(value) == 0 {
			return "", fmt.Errorf("missing Secret/%s key %s in namespace %s", env.SecretName, key, env.SecretNamespace)
		}
		return string(value), nil
	}

	optional := func(key string) string {
		value := secret.Data[key]
		if len(value) == 0 {
			return ""
		}
		return string(value)
	}

	e2eSso, err := require("E2E_SSO_PASSWORD")
	if err != nil {
		return err
	}
	_ = os.Setenv("E2E_SSO_PASSWORD", e2eSso)

	owner, err := require("PLAYWRIGHT_OWNER_PASSWORD")
	if err != nil {
		return err
	}
	viewer, err := require("PLAYWRIGHT_VIEWER_PASSWORD")
	if err != nil {
		return err
	}
	newMember, err := require("PLAYWRIGHT_NEWMEMBER_PASSWORD")
	if err != nil {
		return err
	}
	_ = os.Setenv("PLAYWRIGHT_OWNER_PASSWORD", owner)
	_ = os.Setenv("PLAYWRIGHT_VIEWER_PASSWORD", viewer)
	_ = os.Setenv("PLAYWRIGHT_NEWMEMBER_PASSWORD", newMember)

	if v := optional("E2E_OWNER_PASSWORD"); v != "" {
		_ = os.Setenv("E2E_OWNER_PASSWORD", v)
	}
	if v := optional("E2E_VIEWER_PASSWORD"); v != "" {
		_ = os.Setenv("E2E_VIEWER_PASSWORD", v)
	}
	if v := optional("E2E_NEWMEMBER_PASSWORD"); v != "" {
		_ = os.Setenv("E2E_NEWMEMBER_PASSWORD", v)
	} else {
		_ = os.Setenv("E2E_NEWMEMBER_PASSWORD", newMember)
	}
	return nil
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3_inner_env.go`

```go
package innerloop

import (
	"encoding/json"
	"fmt"
	"os"
	"regexp"
	"strings"
)

func loadTelepresenceEnvAllowlist(path string) (map[string]string, error) {
	b, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	var raw map[string]any
	if err := json.Unmarshal(b, &raw); err != nil {
		return nil, err
	}

	ident := regexp.MustCompile(`^[A-Za-z_][A-Za-z0-9_]*$`)
	// Update (648): no NEXTAUTH_/NEXT_PUBLIC_ allowlisting for cluster runtime; Auth.js v5 uses AUTH_*
	allow := regexp.MustCompile(`^(AUTH_|AMEIDE_|REDIS_|INFERENCE_|OTEL_|LOG_LEVEL$|DEPLOYMENT_ENVIRONMENT$|ENVIRONMENT$|PORT$|HOSTNAME$|SERVICE_VERSION$|NEXT_TELEMETRY_DISABLED$)`)
	out := map[string]string{}
	for k, v := range raw {
		if !ident.MatchString(k) || !allow.MatchString(k) {
			continue
		}
		out[k] = fmt.Sprintf("%v", v)
	}
	return out, nil
}

func envMapFromOS() map[string]string {
	out := map[string]string{}
	for _, kv := range os.Environ() {
		k, v, ok := strings.Cut(kv, "=")
		if !ok {
			continue
		}
		out[k] = v
	}
	return out
}

func envSliceFromMap(m map[string]string) []string {
	out := make([]string, 0, len(m))
	for k, v := range m {
		out = append(out, k+"="+v)
	}
	return out
}


```

## `packages/ameide_core_cli/internal/innerloop/phase3_preflight.go`

```go
package innerloop

import (
	"context"
	"crypto/tls"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/url"
	"strings"
	"time"
)

func assertNonLoopbackAndReachable(rawURL string) error {
	u, err := url.Parse(rawURL)
	if err != nil {
		return fmt.Errorf("invalid URL %q: %w", rawURL, err)
	}
	host := u.Hostname()
	port := u.Port()
	if port == "" {
		if u.Scheme == "https" {
			port = "443"
		} else {
			port = "80"
		}
	}

	addrs, err := net.LookupIP(host)
	if err != nil || len(addrs) == 0 {
		return fmt.Errorf("DNS lookup failed for %s", host)
	}
	for _, ip := range addrs {
		if ip.IsLoopback() {
			return fmt.Errorf("%s resolves to loopback (%s); fix cluster wiring (prefer FQDN/ExternalName) rather than local host overrides", host, ip.String())
		}
	}

	c, err := net.DialTimeout("tcp", net.JoinHostPort(host, port), 3*time.Second)
	if err != nil {
		return fmt.Errorf("cannot reach %s:%s (TCP connect failed)", host, port)
	}
	_ = c.Close()
	return nil
}

func waitHTTPReady(url string, tries int, delay time.Duration) error {
	client := &http.Client{Timeout: 2 * time.Second}
	for i := 0; i < tries; i++ {
		resp, err := client.Get(url)
		if err == nil && resp != nil {
			_, _ = io.Copy(io.Discard, resp.Body)
			_ = resp.Body.Close()
			if resp.StatusCode >= 200 && resp.StatusCode < 400 {
				return nil
			}
		}
		time.Sleep(delay)
	}
	return context.DeadlineExceeded
}

func ingressPreflight(baseURL, agentID string) error {
	u := strings.TrimRight(baseURL, "/") + "/api/auth/providers"
	tr := &http.Transport{TLSClientConfig: &tls.Config{InsecureSkipVerify: true}}
	client := &http.Client{Transport: tr, Timeout: 10 * time.Second}
	req, _ := http.NewRequest("GET", u, nil)
	req.Header.Set("X-Ameide-Agent", agentID)
	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("ingress request with routing header failed: %w (%s)", err, u)
	}
	defer resp.Body.Close()
	_, _ = io.Copy(io.Discard, resp.Body)
	if resp.StatusCode != 200 {
		return fmt.Errorf("ingress request with routing header returned HTTP %d (%s)", resp.StatusCode, u)
	}
	return nil
}


```

## `packages/ameide_core_cli/internal/innerloop/phase3_nextdev.go`

```go
package innerloop

import (
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

func startLoggedProcess(env map[string]string, repoRoot, logFile, command string, args ...string) (func() error, error) {
	if err := requireCmd(command); err != nil {
		return nil, err
	}
	if err := EnsureDir0700(filepath.Dir(logFile)); err != nil {
		return nil, err
	}
	f, err := os.OpenFile(logFile, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0o600)
	if err != nil {
		return nil, err
	}

	cmd := exec.Command(command, args...)
	cmd.Dir = repoRoot
	cmd.Env = append(os.Environ(), envSliceFromMap(env)...)
	cmd.Stdout = f
	cmd.Stderr = f
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
	if err := cmd.Start(); err != nil {
		_ = f.Close()
		return nil, err
	}

	pgid := cmd.Process.Pid
	waitDone := make(chan struct{})
	go func() {
		_ = cmd.Wait()
		_ = f.Close()
		close(waitDone)
	}()

	kill := func() error {
		_ = syscall.Kill(-pgid, syscall.SIGTERM)
		select {
		case <-waitDone:
			return nil
		case <-time.After(1 * time.Second):
		}
		_ = syscall.Kill(-pgid, syscall.SIGKILL)
		<-waitDone
		return nil
	}

	return kill, nil
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3_playwright.go`

```go
package innerloop

import (
	"context"
	"encoding/json"
	"errors"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"syscall"
	"time"
)

func runPlaywrightInnerLoop(ctx context.Context, runEnv map[string]string, repoRoot, runRoot, stateRoot string) (int, error) {
	outDir := os.Getenv("PLAYWRIGHT_OUTPUT_DIR")
	if outDir == "" {
		outDir = filepath.Join(runRoot, "test-results", "e2e", "test-results")
	}

	hasFailed, _ := hasLastFailedTests(stateRoot)
	ranLastFailed := false
	if hasFailed {
		ranLastFailed = true
		_ = copyLastRunFiles(stateRoot, outDir)
		logFile := filepath.Join(runRoot, "logs", "phase3", "playwright-last-failed.log")
		rc, err := runPlaywright(ctx, runEnv, repoRoot, logFile, []string{"test", "--last-failed", "--max-failures=1"})
		if err != nil {
			return rc, err
		}
		if rc != 0 && containsNoTestsFound(logFile) {
			rc = 0
		}
		if rc == 0 {
			_ = copyLastRunFiles(outDir, stateRoot)
		}
		if rc != 0 {
			return rc, nil
		}
	}

	if !ranLastFailed && hasGitTrackedChanges(ctx, repoRoot) {
		logFile := filepath.Join(runRoot, "logs", "phase3", "playwright-only-changed.log")
		c, cancel := context.WithTimeout(ctx, 90*time.Second)
		defer cancel()
		rc, err := runPlaywright(c, runEnv, repoRoot, logFile, []string{"test", "--only-changed", "--max-failures=1"})
		if err != nil && errors.Is(err, context.DeadlineExceeded) {
			rc = 0
			err = nil
		} else if err != nil && errors.Is(err, context.Canceled) {
			return 1, err
		}
		if rc != 0 && (rc == 124 || rc == 137) {
			rc = 0
		}
		if rc != 0 && containsNoTestsFound(logFile) {
			rc = 0
		}
		if rc != 0 {
			_ = copyLastRunFiles(outDir, stateRoot)
			return rc, nil
		}
	}

	rc, err := runPlaywright(ctx, runEnv, repoRoot, filepath.Join(runRoot, "logs", "phase3", "playwright-full.log"), []string{"test", "--max-failures=1"})
	_ = copyLastRunFiles(outDir, stateRoot)
	return rc, err
}

func runPlaywright(ctx context.Context, runEnv map[string]string, repoRoot, logFile string, pwArgs []string) (int, error) {
	if err := requireCmd("pnpm"); err != nil {
		return 127, err
	}
	if err := EnsureDir0700(filepath.Dir(logFile)); err != nil {
		return 127, err
	}
	f, err := os.OpenFile(logFile, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0o600)
	if err != nil {
		return 127, err
	}
	defer f.Close()

	args := append([]string{"-C", "services/www_ameide_platform", "exec", "playwright"}, pwArgs...)
	cmd := exec.Command("pnpm", args...)
	cmd.Dir = repoRoot
	cmd.Env = append(os.Environ(), envSliceFromMap(runEnv)...)
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
	cmd.Stdout = io.MultiWriter(os.Stdout, f)
	cmd.Stderr = io.MultiWriter(os.Stderr, f)

	if err := cmd.Start(); err != nil {
		return 1, err
	}

	waitCh := make(chan error, 1)
	go func() { waitCh <- cmd.Wait() }()

	select {
	case err := <-waitCh:
		if err == nil {
			return 0, nil
		}
		var ee *exec.ExitError
		if errors.As(err, &ee) {
			if ws, ok := ee.Sys().(syscall.WaitStatus); ok {
				return ws.ExitStatus(), nil
			}
			return 1, nil
		}
		return 1, err
	case <-ctx.Done():
		pgid := cmd.Process.Pid
		_ = syscall.Kill(-pgid, syscall.SIGTERM)
		select {
		case <-waitCh:
		case <-time.After(2 * time.Second):
			_ = syscall.Kill(-pgid, syscall.SIGKILL)
			<-waitCh
		}
		if errors.Is(ctx.Err(), context.DeadlineExceeded) {
			return 124, ctx.Err()
		}
		return 137, ctx.Err()
	}
}

func hasGitTrackedChanges(ctx context.Context, repoRoot string) bool {
	cmd1 := exec.CommandContext(ctx, "git", "diff", "--quiet", "--no-ext-diff")
	cmd1.Dir = repoRoot
	if err := cmd1.Run(); err != nil {
		return true
	}
	cmd2 := exec.CommandContext(ctx, "git", "diff", "--cached", "--quiet", "--no-ext-diff")
	cmd2.Dir = repoRoot
	return cmd2.Run() != nil
}

func hasLastFailedTests(root string) (bool, error) {
	var errFound = errors.New("found")
	var found bool
	err := filepath.WalkDir(root, func(path string, d os.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() {
			return nil
		}
		if d.Name() != ".last-run.json" {
			return nil
		}
		b, err := os.ReadFile(path)
		if err != nil {
			return nil
		}
		var v struct {
			FailedTests []any `json:"failedTests"`
		}
		if err := json.Unmarshal(b, &v); err == nil && len(v.FailedTests) > 0 {
			found = true
			return errFound
		}
		return nil
	})
	if err != nil && errors.Is(err, errFound) {
		return true, nil
	}
	return found, nil
}

func copyLastRunFiles(srcRoot, destRoot string) error {
	return filepath.WalkDir(srcRoot, func(path string, d os.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() {
			return nil
		}
		if d.Name() != ".last-run.json" {
			return nil
		}
		rel, err := filepath.Rel(srcRoot, path)
		if err != nil {
			return nil
		}
		dest := filepath.Join(destRoot, rel)
		if err := EnsureDir0700(filepath.Dir(dest)); err != nil {
			return nil
		}
		b, err := os.ReadFile(path)
		if err != nil {
			return nil
		}
		_ = os.WriteFile(dest, b, 0o600)
		return nil
	})
}

func containsNoTestsFound(logFile string) bool {
	b, err := os.ReadFile(logFile)
	if err != nil {
		return false
	}
	return strings.Contains(string(b), "Error: No tests found")
}

```

## `packages/ameide_core_cli/internal/innerloop/phase3_timing.go`

```go
package innerloop

import (
	"fmt"
	"sort"
	"strings"
	"time"
)

type stepTimings struct {
	starts map[string]time.Time
	durs   map[string]time.Duration
}

func newStepTimings() *stepTimings {
	return &stepTimings{
		starts: map[string]time.Time{},
		durs:   map[string]time.Duration{},
	}
}

func (s *stepTimings) Start(name string) { s.starts[name] = time.Now() }

func (s *stepTimings) End(name string) {
	if t, ok := s.starts[name]; ok {
		s.durs[name] = time.Since(t)
	}
}

func (s *stepTimings) Summary() string {
	var keys []string
	for k := range s.durs {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	var b strings.Builder
	for _, k := range keys {
		b.WriteString(fmt.Sprintf("[inner-loop-test][phase3] %s=%s\n", k, s.durs[k].Round(time.Millisecond)))
	}
	return b.String()
}


```

## `packages/ameide_core_cli/internal/innerloop/phase3_runner.go`

```go
package innerloop

import (
	"context"
	"fmt"
	"net/url"
	"os"
	"path/filepath"
	"strings"
	"time"
)

func RunPhase3TiltResourceFromEnv(ctx context.Context) error {
	repoRoot, err := FindRepoRoot(ctx)
	if err != nil {
		return err
	}

	env, err := loadPhase3Env(repoRoot)
	if err != nil {
		return err
	}

	logRoot := filepath.Join(env.RunRoot, "logs", "phase3")
	if err := EnsureDir0700(logRoot); err != nil {
		return err
	}
	if err := EnsureDir0700(filepath.Join(env.RunRoot, "test-results", "e2e")); err != nil {
		return err
	}

	if err := requireCmd("pnpm"); err != nil {
		return err
	}
	if err := requireCmd(env.TelepresenceBin); err != nil {
		return err
	}

	kc, err := newKubeClient(ctx, env.KubeContext)
	if err != nil {
		return err
	}

	tp := newTelepresenceClient(env.TelepresenceBin)
	if ok, err := tp.SupportsHTTPHeaderIntercept(ctx); err != nil {
		return err
	} else if !ok {
		return fmt.Errorf("telepresence does not support --http-header intercept filtering (requires Telepresence 2.25+)")
	}

	if err := kc.EnsureWorkloadInjectionEnabled(ctx, env); err != nil {
		return err
	}

	if err := loadPlaywrightSecrets(ctx, kc, env); err != nil {
		return err
	}

	grpcURL, err := kc.ResolveEnvoyGrpcBaseURL(ctx, env.KubeNamespace)
	if err != nil {
		return err
	}
	env.GrpcBaseURL = grpcURL

	ensureNoTelepresenceStaleSession(ctx, tp)
	defer func() {
		_ = os.Remove(env.EnvJSONFile)
		tp.Leave(context.Background(), env.InterceptName)
		tp.QuitStopDaemons(context.Background())
	}()

	self, err := os.Executable()
	if err != nil {
		return err
	}

	connectArgs := []string{"connect", "-n", env.KubeNamespace}
	if env.KubeContext != "" {
		connectArgs = append(connectArgs, "--context", env.KubeContext)
	}
	connectArgs = append(connectArgs, "--", self, "dev", "_inner-loop-test-phase3-connected")

	connectEnv := map[string]string{
		"AGENT_CI_REPO_ROOT":           env.RepoRoot,
		"AGENT_CI_RUN_ROOT":            env.RunRoot,
		"AGENT_CI_TELEPRESENCE_BIN":    env.TelepresenceBin,
		"AGENT_CI_KUBE_NAMESPACE":      env.KubeNamespace,
		"AGENT_CI_KUBE_CONTEXT":        env.KubeContext,
		"AGENT_CI_TELEPRESENCE_TARGET": env.TelepresenceTarget,
		"AGENT_CI_BASE_URL":            env.BaseURL,
		"AMEIDE_AGENT_ID":              env.AgentID,
		"AGENT_CI_INTERCEPT_NAME":      env.InterceptName,
		"AGENT_CI_WORKLOAD":            env.Workload,
		"AGENT_CI_ENV_JSON_FILE":       env.EnvJSONFile,
		"AGENT_CI_GRPC_BASE_URL":       env.GrpcBaseURL,
	}

	e2eRoot := filepath.Join(env.RunRoot, "test-results", "e2e")
	connectEnv["PLAYWRIGHT_JUNIT_OUTPUT_FILE"] = filepath.Join(e2eRoot, "junit.xml")
	connectEnv["PLAYWRIGHT_HTML_REPORT_DIR"] = filepath.Join(e2eRoot, "playwright-report")
	connectEnv["PLAYWRIGHT_OUTPUT_DIR"] = filepath.Join(e2eRoot, "test-results")
	connectEnv["PLAYWRIGHT_TRACE_DIR"] = filepath.Join(e2eRoot, "traces")
	connectEnv["PLAYWRIGHT_SCREENSHOT_DIR"] = filepath.Join(e2eRoot, "screenshots")
	connectEnv["PLAYWRIGHT_VIDEO_DIR"] = filepath.Join(e2eRoot, "videos")
	connectEnv["PLAYWRIGHT_LOCAL_WARM_URL"] = "http://127.0.0.1:3001"
	connectEnv["BASE_URL"] = env.BaseURL
	connectEnv["INTEGRATION_MODE"] = "cluster"

	logFile := filepath.Join(logRoot, "telepresence-connect.log")
	err = RunTee(ctx, ExecSpec{
		Dir:     env.RepoRoot,
		Env:     connectEnv,
		LogFile: logFile,
		Command: env.TelepresenceBin,
		Args:    connectArgs,
	})
	if err != nil {
		tp.GatherLogsZip(ctx, env.RunRoot)
		return err
	}
	return nil
}

func RunPhase3ConnectedFromEnv(ctx context.Context) error {
	env := phase3Env{
		RepoRoot:        os.Getenv("AGENT_CI_REPO_ROOT"),
		RunRoot:         os.Getenv("AGENT_CI_RUN_ROOT"),
		TelepresenceBin: os.Getenv("AGENT_CI_TELEPRESENCE_BIN"),
		KubeNamespace:   os.Getenv("AGENT_CI_KUBE_NAMESPACE"),
		KubeContext:     os.Getenv("AGENT_CI_KUBE_CONTEXT"),
		BaseURL:         os.Getenv("AGENT_CI_BASE_URL"),
		AgentID:         os.Getenv("AMEIDE_AGENT_ID"),
		InterceptName:   os.Getenv("AGENT_CI_INTERCEPT_NAME"),
		Workload:        os.Getenv("AGENT_CI_WORKLOAD"),
		EnvJSONFile:     os.Getenv("AGENT_CI_ENV_JSON_FILE"),
		GrpcBaseURL:     os.Getenv("AGENT_CI_GRPC_BASE_URL"),
	}
	if env.RepoRoot == "" || env.RunRoot == "" || env.TelepresenceBin == "" || env.KubeNamespace == "" || env.InterceptName == "" || env.Workload == "" || env.EnvJSONFile == "" || env.AgentID == "" {
		return fmt.Errorf("phase3-connected missing required AGENT_CI_* env vars")
	}

	tp := newTelepresenceClient(env.TelepresenceBin)
	tp.WriteStatusArtifact(ctx, env.RunRoot)
	tmVer, err := tp.TrafficManagerVersion(ctx)
	if err != nil {
		return err
	}
	if !semverGE(tmVer, "2.25.0") {
		return fmt.Errorf("traffic-manager %s is < 2.25.0 (required for HTTP header intercept filtering)", tmVer)
	}

	tp.Leave(ctx, env.InterceptName)

	self, err := os.Executable()
	if err != nil {
		return err
	}

	interceptArgs := []string{
		"intercept", env.InterceptName,
		"--workload", env.Workload,
		"--mechanism", "http",
		"--port", "3001:http",
		"--http-header", "X-Ameide-Agent=" + env.AgentID,
		"--env-json", env.EnvJSONFile,
		"--mount=false",
		"--", self, "dev", "_inner-loop-test-phase3-inner",
	}

	innerEnv := map[string]string{
		"AGENT_CI_REPO_ROOT":      env.RepoRoot,
		"AGENT_CI_RUN_ROOT":       env.RunRoot,
		"AGENT_CI_ENV_JSON_FILE":  env.EnvJSONFile,
		"AGENT_CI_GRPC_BASE_URL":  env.GrpcBaseURL,
		"AGENT_CI_KUBE_NAMESPACE": env.KubeNamespace,
		"BASE_URL":                env.BaseURL,
		"INTEGRATION_MODE":        "cluster",
	}

	logFile := filepath.Join(env.RunRoot, "logs", "phase3", "telepresence-intercept.log")
	return RunTee(ctx, ExecSpec{
		Dir:     env.RepoRoot,
		Env:     innerEnv,
		LogFile: logFile,
		Command: env.TelepresenceBin,
		Args:    interceptArgs,
	})
}

func RunPhase3InnerFromEnv(ctx context.Context) error {
	repoRoot := os.Getenv("AGENT_CI_REPO_ROOT")
	runRoot := os.Getenv("AGENT_CI_RUN_ROOT")
	envJSON := os.Getenv("AGENT_CI_ENV_JSON_FILE")
	grpcBase := os.Getenv("AGENT_CI_GRPC_BASE_URL")
	baseURL := os.Getenv("BASE_URL")
	agentID := os.Getenv("AMEIDE_AGENT_ID")

	if repoRoot == "" || runRoot == "" || envJSON == "" || grpcBase == "" {
		return fmt.Errorf("phase3-inner missing required AGENT_CI_* env vars")
	}

	logRoot := filepath.Join(runRoot, "logs", "phase3")
	if err := EnsureDir0700(logRoot); err != nil {
		return err
	}

	step := newStepTimings()
	defer func() {
		_ = WriteFile0600(filepath.Join(logRoot, "timings.txt"), []byte(step.Summary()))
		fmt.Fprintln(os.Stdout, "[inner-loop-test][phase3] substep timings")
		fmt.Fprint(os.Stdout, step.Summary())
	}()

	envAllowlisted, err := loadTelepresenceEnvAllowlist(envJSON)
	if err != nil {
		return err
	}

	runEnv := envMapFromOS()
	for k, v := range envAllowlisted {
		runEnv[k] = v
	}

	runEnv["NODE_ENV"] = "development"
	runEnv["AMEIDE_FORCE_NODE_ENV"] = "development"
	runEnv["OTEL_SDK_DISABLED"] = "true"
	runEnv["OTEL_TRACES_EXPORTER"] = "none"
	runEnv["OTEL_METRICS_EXPORTER"] = "none"
	runEnv["OTEL_LOGS_EXPORTER"] = "none"

	if runEnv["AUTH_COOKIE_DOMAIN"] == "" && baseURL != "" {
		if d := cookieDomainFromBaseURL(baseURL); d != "" {
			runEnv["AUTH_COOKIE_DOMAIN"] = d
		}
	}

	if shouldOverrideBaseURL(runEnv["AMEIDE_GRPC_BASE_URL"]) {
		runEnv["AMEIDE_GRPC_BASE_URL"] = grpcBase
	}
	if shouldOverrideBaseURL(runEnv["AMEIDE_ENVOY_URL"]) {
		runEnv["AMEIDE_ENVOY_URL"] = grpcBase
	}
	if runEnv["AMEIDE_REDIS_FORCE_DIRECT"] == "" {
		runEnv["AMEIDE_REDIS_FORCE_DIRECT"] = "1"
	}

	step.Start("grpc_preflight")
	if err := assertNonLoopbackAndReachable(runEnv["AMEIDE_GRPC_BASE_URL"]); err != nil {
		step.End("grpc_preflight")
		return err
	}
	step.End("grpc_preflight")

	serverLog := filepath.Join(logRoot, "www-ameide-platform-dev.log")
	_ = os.Remove(serverLog)

	step.Start("next_dev_start")
	kill, err := startLoggedProcess(runEnv, repoRoot, serverLog, "pnpm", "-C", "services/www_ameide_platform", "dev")
	if err != nil {
		step.End("next_dev_start")
		return err
	}
	defer func() { _ = kill() }()
	step.End("next_dev_start")

	step.Start("next_dev_ready")
	if err := waitHTTPReady("http://127.0.0.1:3001/api/auth/providers", 240, 500*time.Millisecond); err != nil {
		step.End("next_dev_ready")
		return fmt.Errorf("www-ameide-platform dev server did not become ready (see %s)", serverLog)
	}
	step.End("next_dev_ready")

	if baseURL != "" && agentID != "" {
		step.Start("ingress_preflight")
		if err := ingressPreflight(baseURL, agentID); err != nil {
			step.End("ingress_preflight")
			return err
		}
		step.End("ingress_preflight")
	}

	stateRoot := filepath.Join(repoRoot, "artifacts", "agent-ci", "_state", "playwright")
	_ = EnsureDir0700(stateRoot)

	step.Start("playwright")
	rc, err := runPlaywrightInnerLoop(ctx, runEnv, repoRoot, runRoot, stateRoot)
	step.End("playwright")
	if err != nil {
		return err
	}
	if rc != 0 {
		return fmt.Errorf("playwright exited with %d", rc)
	}
	return nil
}

func cookieDomainFromBaseURL(base string) string {
	parsed, err := url.Parse(base)
	if err != nil {
		return ""
	}
	host := parsed.Hostname()
	if host == "" {
		return ""
	}
	if strings.HasPrefix(host, "platform.") {
		return "." + strings.TrimPrefix(host, "platform.")
	}
	parts := strings.Split(host, ".")
	if len(parts) >= 3 {
		return "." + strings.Join(parts[1:], ".")
	}
	return ""
}

func shouldOverrideBaseURL(value string) bool {
	if strings.TrimSpace(value) == "" {
		return true
	}
	parsed, err := url.Parse(value)
	if err != nil {
		return true
	}
	host := parsed.Hostname()
	if host == "" {
		return true
	}
	if host == "localhost" || host == "0.0.0.0" || host == "::1" || strings.HasPrefix(host, "127.") {
		return true
	}
	if !strings.Contains(host, ".") {
		return true
	}
	return false
}

```
