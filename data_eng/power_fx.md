// when module changes, store module and clear downstream state
Set(varModuleIdSafe, cmbSelectModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbSelectModule.Selected.ModuleName); // or .Title depending on your schema

Reset(cmbSelectCategory);
Clear(colModuleCatItems_Working);   // <-- leave working list empty until a category is chosen
