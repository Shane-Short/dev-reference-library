// ---- Session-scoped globals ----
Set(varSelectedModuleId, Blank());
Set(varSelectedModuleName, "");

// Where we store the *current active* CatItem_IDs for the selected module (text ids)
ClearCollect(colRefActiveIds); // shape: [ { id: "catitem-guid" }, ... ]

// Where we store *pending toggles* across categories for the selected module
// shape: [ { Mod_ID: "<module-id>", CatItem_ID: "<catitem-id>", Desired: true/false }, ... ]
ClearCollect(colToggleLog);

// Where we store *pending skill-type edits* across categories
// shape: [ { CatItem_ID: "<catitem-id>", NewType: "<Basic|Special|Advanced>" }, ... ]
ClearCollect(colTypeEdits);

// Working gallery data for the *currently picked* category
// (we’ll populate this in Step 2 when we wire the category picker)
ClearCollect(colModuleCatItems_Working);




// 1) Cache the selected module id + name
Set(varSelectedModuleId, cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// 2) Snapshot active Reference rows for this module (normalize CatItem_ID to text)
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && IsActive = true
            ),
            CatItem_ID
        ),
        id, Text(CatItem_ID)
    )
);

// 3) Clear any old pending toggles/edits for the *previous* module
Clear(colToggleLog);
Clear(colTypeEdits);

// 4) If a category is already chosen, we’ll rebuild the gallery in Step 2.
// For now do nothing here.







