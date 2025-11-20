def build_pm_flex_path(config: dict) -> tuple[Path, str]:
    """
    Build the full path to the PM_Flex.csv file for the current customer WW.

    Parameters
    ----------
    config : dict
        Parsed config.yaml.

    Returns
    -------
    (file_path, ww_str) : tuple[Path, str]
        Path to PM_Flex.csv and the WW string used (e.g. '2025WW47').
    """
    src_cfg = config["pm_flex_source"]

    root_path = Path(src_cfg["root_path"])
    file_name = src_cfg.get("file_name", "PM_Flex.csv")

    ww_str = get_current_customer_ww()
    ww_folder = root_path / ww_str
    file_path = ww_folder / file_name

    if not file_path.exists():
        msg = f"PM_Flex file not found at {file_path}"
        logger.error(msg)
        raise FileNotFoundError(msg)

    logger.info("Using PM_Flex file from WW %s at: %s", ww_str, file_path)
    return file_path, ww_str
