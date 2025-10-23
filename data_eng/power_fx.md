If(
    IsBlank(varLastSaved),
    "Last saved: —",
    "Last saved: " &
    Text(
        varLastSaved,
        "[$-en-US]mmm d, yyyy h:mm tt",
        "en-US",
        "Pacific Standard Time"
    )
)
