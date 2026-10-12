# Robot

`humanoid/vega_1/vega_1_gripper.mjcf.xml` is the Dexmate Vega 1 Pro bimanual mobile manipulator with grippers, as simulated in MeMoREval.

```
robots/
  humanoid/vega_1/
    vega_1_gripper.mjcf.xml
    meshes/mjcf/            visual and convex collision meshes
    meshes/collision/       collision meshes
  hands/dexd_gripper/meshes/collision/   gripper collision meshes referenced by the MJCF
```

Mesh paths in the MJCF are relative, so keep this layout when copying the model.