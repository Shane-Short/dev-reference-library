if(
  equals(length(variables('varPresetIdsForAdds')), 0),
  "Team_Preset_ID eq '' and IsActive eq 1",
  concat(
    "(",
    concat(
      "Team_Preset_ID eq '",
      concat(
        join(variables('varPresetIdsForAdds'), "' or Team_Preset_ID eq '"),
        "'"
      )
    ),
    ") and IsActive eq 1"
  )
)
