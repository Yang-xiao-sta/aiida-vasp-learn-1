# aiida-vasp-learn-1
---
file_format: mystnb
kernelspec:
  display_name: Python 3
  name: python3

myst:
  substitutions:
    VaspWorkChain: "{py:class}`VaspWorkChain <aiida_vasp.workchains.v2.vasp.VaspWorkChain>`"
    VaspCalculation: "{py:class}`VaspCalculation <aiida_vasp.calcs.vasp.VaspCalculation>`"
    VaspBandsWorkchain: "{py:class}`VaspWorkChain <aiida_vasp.workchains.v2.bands.VaspBandsWorkchain>`"
    VaspRelaxWorkChain: "{py:class}`VaspWorkChain <aiida_vasp.workchains.v2.relax.VaspRelaxWorkChain>`"
    VaspConvergenceWorkChain: "{py:class}`VaspWorkChain <aiida_vasp.workchains.v2.converge.VaspConvergenceWorkChain>`"
    calcfunction: "{py:class}`calcfunction <aiida.engine.calcfunction>`"
    workfunction: "{py:class}`calcfunction <aiida.engine.workfunction>`"
    VaspInputGenerator: "{py:class}`VaspInputGenerator <aiida_vasp.protocols.generator.VaspInputGenerator>`"
    PresetConfig: "{py:class}`PresetConfig <aiida_vasp.protocols.generator.PresetConfig>`"
---

(workflow_inputs)=

# How to setup calculations

## How to know what inputs are allowed for a calculation/workchain

AiiDA workchains define their inputs using *input ports* and *port namespaces*. Each workchain exposes a set of input ports (such as `structure`, `parameters`, `kpoints`, etc.), and these can be grouped into namespaces for logical organization (e.g., `parameters.incar`). This structure allows for flexible and hierarchical input definitions, making it easier to manage complex workflows.

The easiest way to explore what inputs a workchain can take is to use the `ProcessBuilder` with tab completion

```python
from aiida.plugin import WorkFlowFactory
builder = WorkflowFactory('vasp.vasp').get_builder()
builder.<tab>
```

Alternatively, one can use `verdi` commandline tool to inspect a workchain:

```bash
verdi plugin list aiida.workflows vasp.vasp
```

The third way is to look into the source code, for example, the  `VaspWorkChain` has the following lines :

```python
class VaspWorkChain:

    @classmethod
    def define(cls, spec: ProcessSpec) -> None:
        super(VaspWorkChain, cls).define(spec)
        spec.expose_inputs(cls._process_class, exclude=('metadata',))
        spec.expose_inputs(
            cls._process_class, namespace='calc', include=('metadata',), namespace_options={'populate_defaults': True}
        )

        # Use a custom validator for backward compatibility
        # This needs to be removed in the next major release/formalized workchain interface
        spec.inputs.validator = validate_calc_job_custom
        spec.inputs['calc']['metadata']['options']['resources']._required = False

        spec.input('kpoints', valid_type=orm.KpointsData, required=False)
        spec.input(
            'potential_family',
            valid_type=orm.Str,
            required=True,
            serializer=to_aiida_type,
            validator=potential_family_validator,
        )
```

In the code above, the `spec.input` call define a series of ports that a calculation may take as inputs.
An input port may contain certain default value, and it may or may not be *required* by a calculation.
A more advanced usage is the `spec.expose_inputs` call, which **expose** existing input ports of another calculation/workchain to the current workchain.
In above, the inputs of a {{ VaspCalculation }} is exposed at the top level as well as nested in a `calc`.
However, the latter only contains a `metadata` port, which is a special input port that allow defining *options* with request resource and wall-time limits or a  {{ VaspCalculation }}.

:::{admonition} Recommended workflow design pattern
:class: dropdown

This design pattern is due to historical reasons. For new projects, we recommend exposing inputs of a sub-workchain/calculation in full inside a nested namespace instead by default.

For example, the following code exposes *all* ports of a {{ VaspCalculation }} except the `kpoints` port:

```python
class UserWorkChain:

    @classmethod
    def define(cls, spec: ProcessSpec) -> None:
        super(VaspWorkChain, cls).define(spec)
        spec.expose_inputs(
            cls._process_class, namespace='calc',
            exclude=('kpoints',)
        )

        spec.input('kpoints', valid_type=orm.KpointsData, required=False)
        spec.input('kpoints_spacing', valid_type=orm.Float, required=False)
        spec.input(
            'potential_family',
            valid_type=orm.Str,
            required=True,
            serializer=to_aiida_type,
            validator=potential_family_validator,
        )
```

The `kpoints` port is exposed at the top level, but when `self.exposed_inputs` is called with `agglomerate=True` (default), the parent namspace is searched with the defined ports.
Hence the  `kpoints` port at the top-level
will be gathered as the input for a {{ VaspCalculation }} automatically.

As an example, if one defines the inputs to a `UserWorkChain` like:
```
UserWorkChain
|- kpoints
|- potential_family
|- calc
    |-parameters
```

Calling `UserWorkChain.exposed_inputs(VaspWorkChain, 'calc', agglomerate=True)` will have the following
inputs gathered and ready to be passed to a {{ VaspCalculation }}

```
VaspCalculation
|- kpoints
|- potentials
|- parameters
|- ...
```

:::

## How to setup inputs of a calculation or a workflow

In this document we will learn how to pass necessary input to the calculation and workflows provided by aiida-vasp.

The input and outputs of the workflows as implemented as `WorkChain` are AiiDA's `Data` types.
The `Data` is a subclass of the `Node` class, which represents data that is stored in the database
and could be used by other  `Process` nodes.
A `WorkChain` has a set of pre-defined input and output ports (which can be dynamic, if needed) that
specifies the types of data that can be passed to and from it.

Some python native types (`float`, `dict`, `str`) have their `Data` counterparts, such as `Float`, `Dict`, `Str` - they can be used as inputs to the workflows directly, but the conversion still takes
place internally.

There are tree ways to pass inputs to the workflows. The most general way is to pass a `dict` object
contains key-values pairs of the data to be passed to each input port of the workchain.

### Using a `dict` object as inputs to a `Process`

```python
from aiida.engine import submit

submit(Process, **inputs)
```

where `Process` is a class of the process to be launched. In aiida-vasp, it may be `VaspCalculation`, `VaspWorkChain` or other provided processes.
The `inputs` is a dictionary containing a nested key-value pairs defining inputs for each port of the process.
A typical `inputs` dictionary for `VaspWorkChain` looks like

```python
inputs = {
  'structure': si_structure,  # An instance of aiida.orm.StructureData
```

or a compact format with `*` indicating repeats:

```
MAGMOM = 0.1*1 1.0*1 5.0*2
```

Very often, we want the initial magnetic moments to be specie-dependent.

We can of course explictly set the `MAGMOM` tags programatically::

```
config = {'O': 0.0, 'Fe': 5.0}
builder = VaspWorkChain.get_builder()
builder.structure = structure
builder.parameters = {
  'incar': {
    ....,
    'magmom': [config.get(site.kind_name, 0.6) for site in structure.sites]
  }
}
```

This will assign the initial magnetic moment of all `Fe` atoms to be 5.0 and `O` atoms to be 0.0, other species will have 0.6 as the initial magnetic moment.

In fact, {{ VaspWorkChain }} provides an input port to do exactly this.
The above code is equivalent to:

```python
config = {'O': 0.0, 'Fe': 5.0, 'default': 0.6}
builder = VaspWorkChain.get_builder()
builder.magmom_mapping = config
```

In addition, the `ISPIN` tag will be set to `2` if it is not explicitly defined in the input.

:::{note}
At the time of writing, `magmom_mapping` port cannot be used for setting the initial three-dimensional spin for none-collinear spin calculations.
:::

## How configure LDA+U calculations

LDA+U calcualtion in VAPS requires the following tags to be set in INCAR:

* `LDAU`: should be set to `.TRUE.` as an overall switch of +U
* `LDAUTYPE`: determines the type of the LDA+U formulism. Usually `2` is used which uses only `LDAUU`.
* `LDAUU`: sets the $$U$$ value of each specie
* `LDAUJ`: sets the $$J$$ value of each specie. The value of $$U-J$$ is often referred as $$U_{eff}$$ for a sepcific specie. This parameter is only used when `LDAUTYPE` is `1`.
* `LDAUL`: controls which angular momentum channel the +U should be applied for each specie. For *3d* transition metals, the angular momentum channel is `2`. The U is not applied if it is set to `-1`.

For example, to add U on Ni atoms with $$U_eff=6 \mathrm{eV}$$ in NiO, we can set

```
LDAU = .TRUE.
LDAUTYPE = 2
LDAUU = 6.0 0.0
LDAUL = 2 -1
```

This assumes that Ni comes first in the POSCAR and POTCAR, followed by O.

We can explicitly define these tags in the inputs to various workchain.
However, it is easier (and less prone to mistake) to use the `ldau_mapping` port for automatically generating these tags:

```python
config = {'mapping': {'Ni': [2, 6.0], 'Fe': [2, 4.0]}}
builder = VaspWorkChain.get_builder()
builder.structure = structure
builder.ldau_mapping = config
```

which sets $$U_{eff} = 6.0 \mathrm{eV}$$ for Ni's $$l=2$$ angular momentum channel, e.g. its $$3d$$
Internally, the {{ VaspWorkChain }} uses {py:func}`get_ldau_keys <aiida_vasp.utils.ldau:get_ldau_keys>` function to generate the INCAR tags and the available keys can be found in its docstring.
