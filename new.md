Ready to code?                                                                                                                                                                                                            ↑

 Here is Claude's plan:                                                                                                                                                                                                    ↑
  ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
 Plan: everestctl instance create                                                                                                                                                                                          ↑

 Context                                                                                                                                                                                                                   ↑

   The CLI already has auth login / auth logout (ctl-login / ctl-logout branches). This branch adds instance create — the primary user flow after authentication. The command must discover which database providers (psmdb/PostgreSQL/pxc) and version bundles are available on the cluster, let the user pick one, and call POST /v1/clusters/{cluster}/namespaces/{namespace}/instances. Different providers expose different version bundles and topologies; the command handles all of them uniformly.                                                                                                                                                                ↑

   ---                                                                                                                                                                                                                       ↑
   Architecture Pattern (mirrors auth)
                                                                                                                                                                                                                             ↑
   commands/instance.go             ← group registration under rootCmd
   commands/instance/create.go      ← Cobra command: flags, PreRun (interactive), Run (delegates)                                                                                                                            ↑
   pkg/cli/instance/create.go       ← business logic: InstanceCreator.Run()
   pkg/cli/flags.go                 ← add FlagInstance* constants                                                                                                                                                            ↑

   ---                                                                                                                                                                                                                       ↑
   Files to Create / Modify
                                                                                                                                                                                                                             ↑
   1. pkg/cli/flags.go (modify)
                                                                                                                                                                                                                             ↑
   Add a // 'instance' flags section:
                                                                                                                                                                                                                             ↑
   FlagInstanceName         = "name"
   FlagInstanceNamespace    = "namespace"                                                                                                                                                                                    ↑
   FlagInstanceProvider     = "provider"
   FlagInstanceCluster      = "cluster"                                                                                                                                                                                      ↑
   FlagInstanceVersion      = "version"        // empty = auto-select default bundle
   FlagInstanceTopology     = "topology"       // empty = auto-select first topology                                                                                                                                         ↑
   FlagInstanceStorageSize  = "storage-size"   // required, e.g. "25Gi"
   FlagInstanceStorageClass = "storage-class"  // optional                                                                                                                                                                   ↑
   FlagInstanceReplicas     = "replicas"       // optional, 0 = omit
                                                                                                                                                                                                                             ↑
   ---
   2. pkg/cli/instance/create.go (new)                                                                                                                                                                                       ↑

   Package instance. No Cobra imports.                                                                                                                                                                                       ↑

   Types:                                                                                                                                                                                                                    ↑

   type Config struct { Pretty bool }                                                                                                                                                                                        ↑

   type CreateOptions struct {                                                                                                                                                                                               ↑
       Name, Namespace, Provider, Cluster, Version, Topology string
       StorageSize, StorageClass string                                                                                                                                                                                      ↑
       Replicas int32
   }                                                                                                                                                                                                                         ↑

   type InstanceCreator struct { config Config; l *zap.SugaredLogger }                                                                                                                                                       ↑

   func NewInstanceCreator(cfg Config, l *zap.SugaredLogger) *InstanceCreator                                                                                                                                                ↑

   func (ic *InstanceCreator) Run(ctx context.Context, opts CreateOptions, cfgPath string) error:                                                                                                                            ↑

   1. config.Load(cfgPath) → GetCurrentContext() → GetUser() → GetServer() (same as pkg/cli/auth/logout.go lines 39–57). Return error if no active context.                                                                  ↑
   2. client.NewClientWithResponses(normalizeServerURL(srv.URL)) with bearerToken(usr.AccessToken).
   3. Resolve version: GetProviderWithResponse(ctx, opts.Cluster, opts.Provider, token) → check HTTP 200 → iterate Spec.Versions, find entry where *Default == true; use its Name. If none marked default, use Versions[0].Name. If empty, leave version string empty.
   4. Resolve topology: iterate keys of *Spec.Topologies, sort, take index 0. If empty, omit topology from spec.                                                                                                             ↑
   5. Build instance JSON as map[string]interface{} (avoids anonymous-struct complexity in client.Instance):
   specMap := map[string]interface{}{                                                                                                                                                                                        ↑
       "provider": opts.Provider,
       "components": map[string]interface{}{                                                                                                                                                                                 ↑
           "engine": map[string]interface{}{
               "storage": map[string]interface{}{"size": opts.StorageSize},                                                                                                                                                  ↑
           },
       },                                                                                                                                                                                                                    ↑
   }
   // Conditionally add:                                                                                                                                                                                                     ↑
   if resolvedVersion != ""  { specMap["version"] = resolvedVersion }
   if resolvedTopology != "" { specMap["topology"] = map[string]interface{}{"type": resolvedTopology} }                                                                                                                      ↑
   if opts.StorageClass != "" { engineMap["storage"]["storageClass"] = opts.StorageClass }
 if opts.Replicas > 0      { engineMap["replicas"] = opts.Replicas }                                                                                                                                                       ↑

   payload := map[string]interface{}{                                                                                                                                                                                        ↑
       "metadata": map[string]interface{}{"name": opts.Name},
       "spec": specMap,                                                                                                                                                                                                      ↑
   }
   6. json.Marshal(payload) → CreateInstanceWithBodyWithResponse(ctx, opts.Cluster, opts.Namespace, "application/json", bytes.NewReader(data), token).                                                                       ↑
   7. Check resp.StatusCode() is 200 or 201. On error, return the API error message.
   8. output.Success("Instance %q created in namespace %q", opts.Name, opts.Namespace) if cfg.Pretty.                                                                                                                        ↑

   Private helpers (inline, ~10 lines total — no auth package changes needed):                                                                                                                                               ↑
   func normalizeServerURL(server string) string  // strip trailing slash, append /v1
   func bearerToken(token string) client.RequestEditorFn  // sets Authorization: Bearer header                                                                                                                               ↑

   Reference pkg/cli/auth/login.go:136-144 and pkg/cli/auth/logout.go:108-113 for exact logic.                                                                                                                               ↑

   ---                                                                                                                                                                                                                       ↑
   3. commands/instance/create.go (new)
                                                                                                                                                                                                                             ↑
   Package instance (commands subpackage). Mirrors commands/auth/login.go exactly.
                                                                                                                                                                                                                             ↑
   Flags (registered in init() using cli.FlagInstance* constants):
   - --name, --namespace, --provider — no defaults, prompted interactively if absent                                                                                                                                         ↑
   - --cluster — default "main"
   - --version, --topology — default "" (auto-resolved in business logic, no interactive prompt)                                                                                                                             ↑
   - --storage-size — default "" (prompted if absent, suggest "25Gi")
   - --storage-class — default "", optional                                                                                                                                                                                  ↑
   - --replicas — int32, default 0, optional
                                                                                                                                                                                                                             ↑
   createPreRun(cmd, _):
                                                                                                                                                                                                                             ↑
   Sets createCfg.Pretty. Then handles interactive prompts for fields the user must provide but didn't flag:
                                                                                                                                                                                                                             ↑
   1. Build API client from current config (same pattern as Run: load config, get context/user/server, construct client). If not logged in, print error and exit.
   2. Namespace (createOpts.Namespace == ""):                                                                                                                                                                                ↑
     - Call c.ListNamespacesWithResponse(ctx, opts.Cluster, token)
     - Print numbered list to stdout: "Available namespaces:\n  1. foo\n  2. bar"                                                                                                                                            ↑
     - tui.NewInput(ctx, "Namespace") with ValidateInputFunc that checks value is in the list
     - Assign to createOpts.Namespace                                                                                                                                                                                        ↑
   3. Provider (createOpts.Provider == ""):
     - Call c.ListProvidersWithResponse(ctx, opts.Cluster, token)                                                                                                                                                            ↑
     - Extract names: (*resp.JSON200.Items)[i].Metadata["name"].(string) — also print alongside display name/kind if available
     - Print numbered list, prompt with validation                                                                                                                                                                           ↑
     - Assign to createOpts.Provider
   4. Name (createOpts.Name == ""):                                                                                                                                                                                          ↑
     - tui.NewInput(ctx, "Instance name") with DNS-label validation (lowercase alphanum + hyphens, max 63 chars, no leading/trailing hyphen). Implement as a validateInstanceName(s string) error function.
     - Assign to createOpts.Name                                                                                                                                                                                             ↑
   5. Storage size (createOpts.StorageSize == ""):
     - tui.NewInput(ctx, "Storage size", tui.WithInputDefaultValue("25Gi")) — optionally add hint "Kubernetes quantity, e.g. 10Gi"                                                                                           ↑
     - Assign to createOpts.StorageSize
   6. Version and topology: no prompts — left empty for auto-resolution in Run.                                                                                                                                              ↑

   createRun(cmd, _):                                                                                                                                                                                                        ↑

   cfgPath, err := config.DefaultPath(); // err → os.Exit(1)                                                                                                                                                                 ↑
   ic := instancecli.NewInstanceCreator(*createCfg, logger.GetLogger())
   if err := ic.Run(cmd.Context(), *createOpts, cfgPath); err != nil {                                                                                                                                                       ↑
       output.PrintError(err, logger.GetLogger(), createCfg.Pretty)
       os.Exit(1)                                                                                                                                                                                                            ↑
   }
                                                                                                                                                                                                                             ↑
   ---
   4. commands/instance.go (new)                                                                                                                                                                                             ↑

   Package commands. Mirrors commands/auth.go exactly:                                                                                                                                                                       ↑

   var instanceCmd = &cobra.Command{                                                                                                                                                                                         ↑
       Use:  "instance <command> [flags]",
       Short: "Manage Everest database instances",                                                                                                                                                                           ↑
       RunE: func(cmd *cobra.Command, _ []string) error { return cmd.Help() },
   }                                                                                                                                                                                                                         ↑

   func init() {                                                                                                                                                                                                             ↑
       rootCmd.AddCommand(instanceCmd)
       instanceCmd.AddCommand(instance.GetCreateCmd())                                                                                                                                                                       ↑
   }
                                                                                                                                                                                                                             ↑
   Import: "github.com/openeverest/openeverest/v2/commands/instance"
                                                                                                                                                                                                                             ↑
   ---
   Key Type References                                                                                                                                                                                                       ↑

   - Generated client: client/crds.gen.go — Provider (line 3193), ProviderList (3292), Instance (1081)                                                                                                                       ↑
   - Storage size union: client.Instance_Spec_Components_Storage_Size — bypassed entirely by using map[string]interface{} approach
   - Generated methods: client/everest-client.gen.go — CreateInstanceWithBodyWithResponse, ListProviders, GetProvider, ListNamespaces                                                                                        ↑
   - Input options (for default value, hint): tui.WithInputDefaultValue, check pkg/cli/tui/input.go for exact option function names
                                                                                                                                                                                                                           ↑
   ---
   Verification

 # Build check
   go build ./...

   # Full non-interactive flow
   everestctl auth login --server http://localhost:8080 --username admin --password admin
   everestctl instance create \
     --name test-mongo \
   --namespace everest \
     --provider psmdb \
     --storage-size 25Gi
Expected: "Instance "test-mongo" created in namespace "everest""

Confirm instance appeared
bectl get instances -n everest

Interactive flow (should prompt for namespace, provider, name, storage-size)
   everestctl instance create
 # Expected: numbered lists printed, prompts appear for each field

   # Error case: bad provider
   everestctl instance create --name x --namespace everest --provider bad --storage-size 25Gi
 # Expected: API error returned cleanly (404 from GetProvider)

   # No auth context
   rm ~/.config/everest/config.yaml
 everestctl instance create --name x --namespace everest --provider psmdb --storage-size 25Gi
   # Expected: "no active context found in config"
