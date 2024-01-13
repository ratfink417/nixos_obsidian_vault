# Enabling Flakes and Selecting Packages

To get started, you'll need to be logged into a freshly installed Nix system. Before we set up our [flake](0 Introduction#Flakes), we'll want and NEED to make some changes to the base configuration file from that comes with nix at [_/etc/nixos/configuration.nix_](0 Introduction#^bd0a6e) The most important thing that needs to be done is we need to enable the [flakes management feature](2 Glossary#^fa85d9) the second thing we need to take care of is figuring out where to add system packages since NixOS only comes with _nano_ for editing files.

Firstly, to take care of the editor and whatever else you want installed on the system, run `sudo nano /etc/nixos/configuration.nix` and move your cursor over to this section of the file.
``` nix
  # List packages installed in system profile. To search, run:
  # $ nix search wget
  environment.systemPackages = with pkgs; [
    #vim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
    #wget
    #git
  ];
```

and change the contents of the square brackets `[]` to include the packages you want installed and if you need to know the name of the package you want, NixOS helpfully provides a website https://mynixos.com/ where you can search not just for a package but the options provided for it by NixOS. My search for vim through gave me results like this.

![[mynixos_com_package_search.png]]
The actual package is _nixpkgs/package/vim_ there were a lot more results but if you were to click on _nixpkgs/option/programs.vim.package_ you'd see some package options you'd be able to set for the vim package. A lot of which are options you set for installing vim, not the vimrc and plugins and all the other crap. All of that crap is still going in dot files in your _$XDG\_HOME_ directory. 

You can search this on the command line as well, 

Next, we need to enable flakes just add this single line at the bottom of your config but still inside the curly braces. The code block below is the last handful of lines at the bottom of the default _configuration.nix_ file
``` nix
  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "23.11"; # Did you read the comment?

  # Enable NixOS flakes management
  nix.settings.experimental-features = ["nix-command" "flakes"];
}
```

The important option we set here is the `nix.settings.experimental-features = ["nix-command" "flakes"];` line. After you're done editing the file, you need to run the command that will generate our new NixOS environment and switch us into it immediately.

``` sh
sudo nixos-build switch
```

Assuming that all went as planned, we can now move on to creating our first flake.
# Creating The Flake
Below is a simple example of a nix flake that's not really doing anything but importing our _configuration.nix_ file as a module, setting our system type and telling our package manager to pull it's packages from the nixpkgs unstable branch.

```nix
{
  description = "My first flake";
  inputs = {
    nixpkgs.url = "nixpkgs/nixos-unstable";
  };
  outputs = { self, nixpkgs, ...}:
    let
      lib = nixpkgs.lib;
    in {
      nixosConfigurations = {

      nixos = lib.nixosSystem {
        system = "x86_64-linux";
        modules = [ ./configuration.nix ];
      };
      };
    };
}
```

- **description:** A short one-line description of the flake
- **inputs:** An [[2 Glossary#^3b069a|attribute set]] specifying the dependencies of the flake
- **outputs:** A [[2 Glossary#^999202|function]] that, takes an [[2 Glossary#^3b069a|attribute set]] of output from each of the input flakes give when they are called by the Nix package manager. ^01685b


  **In the example above, `inputs.nixpkgs` contains the result of the call to the `outputs` function of the `nixpkgs` flake.**
## Inputs
The inputs section in our example contains an attribute in short hand form that declares an internal [[2 Glossary#^3b069a|attribute set]] called _nixpkgs_ and sets it's _url_ attribute to contain several pieces of data: a _type_ "**git**" by default for the `nixpkgs` set, an _owner_ "**nixpkgs**" in this case and a _repo_ "**nixos-unstable**". If you actually took a look at this repo, you'd sett that it's a [[2 Glossary#^acc8a0|flake]] with it's own _flake.nix_ file and everything just like we're creating here. So, when we pass it to the [[1 Your First NixOS Flake#^01685b|outputs function]] later and process **nixpkgs** we're referring it to a flake hosted by the NixOS team on Github.

```nix
inputs = {
    nixpkgs.url = "nixpkgs/nixos-unstable";
  };
```

A fully expanded version of the inputs would look like this. The url syntax used above is much easier to read/write but I think it's up to you how you want to structure this... not totally sure. Seems like there could be a need for both when variable paths get super long and you have a lot of stuff to declare.

```nix
inputs = {
	nixpkgs = {
		type = "github";
		owner = "nixpkgs";
		repo = "nixos-unstable";
	};
};
```
## Outputs
The [[1 Your First NixOS Flake#^01685b|outputs function]] from our example flake is a [[2 Glossary#^c9bcfd|set-pattern function declaration]] in this case, the [[2 Glossary#^3b069a|attrset]] used as the argument includes: a [[2 Glossary#^80279d|self]] reference, the [[2 Glossary#^5e70bd|nixpkgs]] repo and an _ellipses_ which just allows _outputs_ to accept more arguments without causing it to fail/error out. 

 ```nix
 outputs = { self, nixpkgs, ...}:
    let
      lib = nixpkgs.lib;
    in {
      nixosConfigurations = {

      nixos = lib.nixosSystem {
        system = "x86_64-linux";
        modules = [ ./configuration.nix ];
      };
      };
    };
```

The next thing it does is bind the **lib** variable to the `nixpkgs.lib` attribute using the `let` and `in` keywords. This means that between the curly braces after `in` that `lib` will refer to `nixpkgs.lib`. This will just make it easier to fill out the attr set returned by _outputs_ so that we don't have to call `nixpkgs.lib.nixosSystem` or whatever other functions we'd want to call from that flake.

We could for example change it out to look like this and not have any problems.
```nix
{
  description = "My first flake";
  inputs = {
    nixpkgs.url = "nixpkgs/nixos-unstable";
  };
  outputs = { self, nixpkgs, ...}:
  {
      nixosConfigurations = {

      nixos = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [ ./configuration.nix ];
      };
      };
  };
}
```

The last thing that's done here is the `nixpkg.lib.nixosSystem` function gets evaluated 











