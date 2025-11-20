def load_pm_flex_df(config: dict) -> pd.DataFrame:
    """
    Load the PM_Flex.csv file for the current customer work week and
    return a raw mirrored DataFrame with a few metadata columns added.

    Columns added:
        - load_ww: customer WW string used (e.g. '2025WW47')
        - source_file: file name of the CSV (e.g. 'PM_Flex.csv')
        - load_ts: ISO 8601 timestamp when the file was loaded (UTC)

    Parameters
    ----------
    config : dict
        Parsed config.yaml.

    Returns
    -------
    df : pd.DataFrame
        PM_Flex data with metadata columns.
    """
    file_path, ww_str = build_pm_flex_path(config)

    # Basic CSV read – if your file needs special options (sep, encoding),
    # we can later pull those from config as well.
    df = pd.read_csv(file_path)

    load_ts = datetime.now(timezone.utc).isoformat()

    df["load_ww"] = ww_str
    df["source_file"] = file_path.name
    df["load_ts"] = load_ts

    logger.info(
        "Loaded PM_Flex file: %s rows from %s (ww=%s)",
        len(df),
        file_path,
        ww_str,
    )

    return df




import yaml
from pathlib import Path

from utils.helpers import load_pm_flex_df

def load_config() -> dict:
    config_path = Path(__file__).resolve().parent / "config" / "config.yaml"
    with open(config_path, "r") as f:
        return yaml.safe_load(f)

if __name__ == "__main__":
    config = load_config()
    df = load_pm_flex_df(config)
    print(df.shape)
    print(df.columns.tolist()[:10])  # show first few column names
    print(df.head(3))
