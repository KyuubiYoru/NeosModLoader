# NeosModLoader Configuration System

NeosModLoader provides a built-in configuration system that can be used to persist configuration values for mods. **This configuration system only exists in NeosModLoader 1.5.0 and later!**

Operations provided:
- Reading value of a config key
- Writing value to a config key
- Enumerating config keys for a mod
- Enumerating mods
- Saving a config to disk

Behind the scenes, configs are saved to a `nml_config` folder in the Neos install directory. The `nml_config` folder contains JSON files, named after each mod dll that defines a config. End users and mod developers do not need to interact with this JSON directly. Mod developers should use the API exposed by NeosModLoader. End users should use interfaces exposed by configuration management mods.

## Working With Your Mod's Configuration

A simple example is below:

```csharp
public class NeosModConfigurationExample : NeosMod
{
    public override string Name => "NeosModConfigurationExample";
    public override string Author => "runtime";
    public override string Version => "1.0.0";
    public override string Link => "https://github.com/zkxs/NeosModConfigurationExample";

    private readonly ModConfigurationKey<int> KEY_COUNT = new ModConfigurationKey<int>("count", "Example counter", internalAccessOnly: true);

    public override ModConfigurationDefinition GetConfigurationDefinition()
    {
        List<ModConfigurationKey> keys = new List<ModConfigurationKey>();
        keys.Add(KEY_COUNT);
        return DefineConfiguration(new Version(1, 0, 0), keys);
    }

    public override void OnEngineInit()
    {
        ModConfiguration config = GetConfiguration();
        int countValue = default(int);
        if (config.TryGetValue(KEY_COUNT, out countValue))
        {
            int oldValue = countValue++;
            Msg($"Incrementing count from {oldValue} to {countValue}");
        }
        else
        {
            Msg($"Initializing count to {countValue}");
        }

        config.Set(KEY_COUNT, countValue);
        config.Save();
    }
}
```

A full example repository that uses a few additional APIs is provided [here](https://github.com/zkxs/NeosModConfigurationExample).

### Defining a Configuration
Mods that wish to use a configuration must define the optional `GetConfigurationDefinition()` method. This method requires you return a `ModConfigurationDefinition` instance, which can only be obtained by calling `DefineConfiguration()`. `DefineConfiguration()` requires two parameters: a version and a list of configuration keys. 

#### Configuration Version
The version should be a [semantic version][semver]—in summary the major version should be bumped for hard breaking changes, and the minor version should be bumped if you break backwards compatibility. NeosModLoader uses this version number to check the saved configuration against your definition and ensure they are compatible.

#### Configuration Keys
Configuration keys define the values your mod's config can store. The relevant class is `ModConfigurationKey<T>`, which has the following constructor:
```csharp
public ModConfigurationKey(string name, string description, Func<T> computeDefault = null, bool internalAccessOnly = false, Predicate<T> valueValidator = null)
```
|Parameter | Description | Default |
| -------- | ----------- | ------- |
| name | Unique name of this config item | *required* |
| description | Human-readable description of this config item | *required* |
| computeDefault | Function that, if present, computes a default value for this key | `null` |
| internalAccessOnly | If true, only the owning mod should have access to this config item. Note that this is *not* enforced by NeosModLoader itself. | `false` |
| valueValidator | A custom function that (if present) checks if a value is valid for this configuration item | `null` |

### Saving the Configuration
Configurations must be saved to disk by calling the `ModConfiguration.Save()` method. NeosModLoader will *not* call this function for you. If you don't call `ModConfiguration.Save()`, your changes will still be available in memory. This allows multiple changes to be batched before you write them all to disk at once. Saving to disk is a relatively expensive operation and should not be performed at high frequency.

### External Changes
The `ModConfiguration` is guaranteed to be the same instance for all calls to `NeosModBase.GetConfiguration()`. This means that other mods may modify the `ModConfiguration` instance you are working with. A `ModConfiguration.TryGetValue()` call will always return the current value for that config item. If you need notice that someone else has changed one of your configs, there are events you can subscribe to. However, the `ModConfiguration.GetValue()` and `TryGetValue()` API is very inexpensive so it is fine to poll.

### Events
The `ModConfiguration` class provides two events you can subscribe to:
- The static event `OnAnyConfigurationChanged` is called if any config value for any mod changed.
- The instance event `OnThisConfigurationChanged` is called if one of the values in this mod's config changed.

Both of these events use the following delegate:
```csharp
public delegate void ConfigurationChangedEventHandler(ConfigurationChangedEvent configurationChangedEvent);
```
A `ConfigurationChangedEvent` has the following properties:
- `ModConfiguration Config` is the configuration the change occurred in
- `ModConfigurationKey Key` is the specific key who's value changed
- `string Label` is a custom label that may be set by whoever changed the configuration. This may be `null`.

### Handling Incompatible Configuration Versions
You can override a `HandleIncompatibleConfigurationVersions()` function in your NeosMod to define how incompatible versions are handled. You have two options:
- `IncompatibleConfigurationHandlingOption.ERROR`: Fail to read the config, and block saving over the config on disk.
- `IncompatibleConfigurationHandlingOption.CLOBBER`: Destroy the saved config and start over from scratch.
- `IncompatibleConfigurationHandlingOption.FORCE_LOAD`: Ignore the version number and load the config anyways. This may throw exceptions and break your mod.

If you do not override `HandleIncompatibleConfigurationVersions()`, the default is to return `ERROR` on all incompatibilities. `HandleIncompatibleConfigurationVersions()` is only called for configs that are detected to be incompatible under [semantic versioning][semver].

Here's an example implementation that can detect mod downgrades and conditionally avoid clobbering your new config:
```csharp
public override IncompatibleConfigurationHandlingOption HandleIncompatibleConfigurationVersions(Version serializedVersion, Version definedVersion)
{
    if (serializedVersion > definedVersion)
    {
        // someone has dared to downgrade my mod
        // this will break the old version instead of nuking my config
        return IncompatibleConfigurationHandlingOption.ERROR;
    }
    else
    {
        // there's an old incompatible config version on disk
        // lets just nuke it instead of breaking
        return IncompatibleConfigurationHandlingOption.CLOBBER;
    }
}
```

### Breaking Changes in Configuration Definition
There are two cases to consider:
- **Forwards Compatible**: Can mod v2 loads config v1?
- **Backwards Compatible**: Can mod v1 loads config v2?

| Action | Forwards Compatible | Backwards Compatible |
| ------ | ------------------- | ---------------------|
| Adding a brand new key | Yes | Yes |
| Removing an existing key | Yes | Yes |
| Adding, altering, or removing a key's default value | Yes | Maybe* |
| Restricting a key's validator | Yes** | Yes |
| Relaxing a key's validator | Yes | Maybe* |
| Changing `internalAccessOnly` to `false` | Yes | Maybe* |
| Changing `internalAccessOnly` to `true` | Yes** |Yes |
| Altering a key's type (removing and re-adding later counts!) | **No** | **No** |

<sup>\* NeosModLoader is compatible, but the old version of your mod's code may not be</sup>  
<sup>\*\* Assuming the new version of your mod properly accounts for reading old configs</sup>

## Working With Other Mods' Configurations
An example of enumerating all configs:
```csharp
void EnumerateConfigs()
{
    IEnumerable<NeosModBase> mods = ModLoader.Mods();
    foreach (NeosModBase mod in mods)
    {
        ModConfiguration config = mod.GetConfiguration();
        if (config != null)
        {
            foreach (ModConfigurationKey key in config.ConfigurationItemDefinitions)
            {
                if (!key.InternalAccessOnly) // while we COULD read internal configs, we shouldn't.
                {
                    if (config.TryGetValue(key, out object value))
                    {
                        Msg($"{mod.Name} has configuration {key.Name} with type {key.ValueType()} and value {value}");
                    }
                    else
                    {
                        Msg($"{mod.Name} has configuration {key.Name} with type {key.ValueType()} and no value");
                    }
                }
            }
        }
    }
}
```
Worth noting here is that this API works with raw untyped objects, because as an external mod you lack the compile-time type information. The API performs its own type checking behind the scenes to prevent incorrect types from being written.

[semver]: https://semver.org/
