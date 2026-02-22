use anyhow::{Context, Result};
use std::fs::{self, File};
use std::path::PathBuf;
use std::process::{Command, Child};
use std::collections::HashMap;
use std::env;
use std::sync::{Arc, Mutex};
use std::time::{SystemTime, UNIX_EPOCH};
use zip::ZipArchive;
use winreg::enums::*;
use winreg::RegKey;
use urlencoding::decode;
use chrono::Utc;
use ctrlc;
use winapi::um::wincon::GetConsoleWindow;
use winapi::um::winuser::ShowWindow;
use crossterm::{
    execute,
    terminal::{SetSize, SetTitle},
};
use std::io::{self, stdout};
use serde::Deserialize;
use discord_rich_presence::{activity, DiscordIpc, DiscordIpcClient};

// i'll try to add a anti-cheat later
// anways, heres my TODO list:
// [x] auto updater
// [x] protocol handler
// [x] download client if not exists
// [x] download studio if not exists
// [x] launch client with args from protocol
// [x] discord rich presence
// [ ] anti-cheat
// [x] better error handling
// [ ] GUI launcher
// Maybe a Linux version in the future? 


const VERSION: &str = "1.2.5";
const EPIKVERSION: &str = "https://www.epik17.xyz/version"; 
const CZIPURL: &str = "https://epikorp.xyz/EPIKPlayerBeta.zip"; 
const SZIPURL: &str = "https://epikorp.xyz/EPIKStudioBeta.zip"; 
const EPIKLAUNCHER: &str = "https://www.epik17.xyz/EPIKLauncherBeta.exe"; 
const DSC_CLIENT_ID: &str = "1233051990582366240";

#[derive(Deserialize, Debug)]
struct Creator {
    id: u64,
    name: String,
    ischeckmarked: bool,
}

#[derive(Deserialize, Debug)]
struct AssetDetails {
    id: u64,
    name: String,
    description: String,
    creator: Creator,
    price: i64,
}

fn appdata() -> Result<PathBuf> {
    let mut p: PathBuf = env::var_os("APPDATA").context("APPDATA environment variable not found")?.into();
    p.push("EPIK17");
    Ok(p)
}

fn cdir() -> Result<PathBuf> {
    let mut p = appdata()?;
    p.push("Client");
    Ok(p)
}

fn sdir() -> Result<PathBuf> {
    let mut p = appdata()?;
    p.push("Studio");
    Ok(p)
}

fn cexe() -> Result<PathBuf> {
    let mut p = cdir()?;
    p.push("EPIKPlayerBeta.exe");
    Ok(p)
}

fn eepikexe() -> Result<PathBuf> {
    let mut target = cdir()?;
    target.push("EPIKPlayerBeta.exe");

    let source = cexe()?;

    if !target.exists() {
        fs::copy(&source, &target)?;
    } else {
        let src_meta = fs::metadata(&source)?;
        let tgt_meta = fs::metadata(&target)?;
        if src_meta.len() != tgt_meta.len() {
            fs::copy(&source, &target)?;
        }
    }

    Ok(target)
}

fn lpath() -> Result<PathBuf> {
    let mut p = appdata()?;
    p.push("EPIKLauncherBeta.exe");
    Ok(p)
}

fn cachelol(url: &str) -> String { 
    let ts = Utc::now().timestamp();
    if url.contains('?') {
        format!("{}&_={}", url, ts)
    } else {
        format!("{}?_={}", url, ts)
    }
}

fn filesdownloader(url: &str, path: &PathBuf) -> Result<()> {
    let client = reqwest::blocking::Client::builder()
        .no_gzip()
        .no_brotli()
        .no_deflate()
        .build()
        .context("Failed to build HTTP client")?;

    let mut attempts = 0;
    loop {
        attempts += 1;
        match do_download(&client, url, path) {
            Ok(_) => break Ok(()),
            Err(e) => {
                if attempts >= 3 {
                    return Err(e).context("Download failed after 3 attempts");
                }
                println!(" > $ Network drop detected, retrying (attempt {}/3)...", attempts + 1);
                std::thread::sleep(std::time::Duration::from_secs(2));
            }
        }
    }
}

fn do_download(client: &reqwest::blocking::Client, url: &str, path: &PathBuf) -> Result<()> {
    let mut r = client.get(url).send()?.error_for_status()?;
    let mut f = File::create(path)?;
    std::io::copy(&mut r, &mut f)?;
    Ok(())
}

fn unzip(path: &PathBuf, out: &PathBuf) -> Result<()> {
    let file = fs::File::open(path)?;
    let mut archive = ZipArchive::new(file)?;

    for i in 0..archive.len() {
        let mut file = archive.by_index(i)?;
        
        let outpath = match file.enclosed_name() {
            Some(path) => out.join(path),
            None => continue,
        };

        if file.name().ends_with('/') {
            fs::create_dir_all(&outpath)?;
        } else {
            if let Some(p) = outpath.parent() {
                if !p.exists() {
                    fs::create_dir_all(p)?;
                }
            }
            let mut outfile = fs::File::create(&outpath)?;
            io::copy(&mut file, &mut outfile)?;
        }

        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            if let Some(mode) = file.unix_mode() {
                fs::set_permissions(&outpath, fs::Permissions::from_mode(mode)).unwrap_or_default();
            }
        }
    }

    Ok(())
}

fn fuckoffprotocol() {
    let hkcu = RegKey::predef(HKEY_CURRENT_USER);
    let _ = hkcu.delete_subkey_all(r"Software\Classes\epik17");
}

fn addprotocol() -> Result<()> {
    println!(" > $ Making the Protocols...");
    let lp = lpath()?;
    let exe = lp.to_str().context("Invalid launcher path")?.to_string();
    let hkcu = RegKey::predef(HKEY_CURRENT_USER);
    let k = hkcu.create_subkey(r"Software\Classes\epik17")?.0;
    k.set_value("", &"URL:epik17 Protocol")?;
    k.set_value("URL Protocol", &"")?;
    k.create_subkey("DefaultIcon")?.0
        .set_value("", &format!("\"{}\",0", exe))?;
    k.create_subkey(r"shell\open\command")?.0
        .set_value("", &format!("\"{}\" \"%1\"", exe))?;
    Ok(())
}

fn create_desktop_shortcut(target_exe: &PathBuf, shortcut_name: &str) -> Result<()> {
    if let Ok(user_profile) = env::var("USERPROFILE") {
        let mut shortcut_path = PathBuf::from(user_profile);
        shortcut_path.push("Desktop");
        shortcut_path.push(format!("{}.lnk", shortcut_name));

        if !shortcut_path.exists() {
            let target_str = target_exe.to_str().unwrap_or("");
            let shortcut_str = shortcut_path.to_str().unwrap_or("");
            
            let ps_script = format!(
                "$wshell = New-Object -ComObject WScript.Shell; $shortcut = $wshell.CreateShortcut('{}'); $shortcut.TargetPath = '{}'; $shortcut.Save()",
                shortcut_str, target_str
            );

            let _ = Command::new("powershell")
                .arg("-NoProfile")
                .arg("-WindowStyle")
                .arg("Hidden")
                .arg("-Command")
                .arg(&ps_script)
                .output()?;
        }
    }
    Ok(())
}

fn updatechecker() -> Result<()> {
    let server_version_req = reqwest::blocking::get(&cachelol(EPIKVERSION)).ok().and_then(|r| r.text().ok());
    let lp = lpath()?;

    let need_launcher = match server_version_req {
        Some(ref v) => v.trim() != VERSION || !lp.exists(),
        None => !lp.exists(),
    };

    if need_launcher {
        if lp.exists() {
            let old_lp = lp.with_extension("old");
            let _ = fs::remove_file(&old_lp);
            let _ = fs::rename(&lp, &old_lp);
        }
        println!(" > $ New launcher version found!");

        filesdownloader(&cachelol(EPIKLAUNCHER), &lp)?;
        println!(" > $ Downloading Launcher...");
        Command::new(&lp).spawn()?;
        std::process::exit(0);
    }

    let need_client = match server_version_req {
        Some(ref v) => v.trim() != VERSION || !cexe()?.exists(),
        None => !cexe()?.exists(),
    };

    if need_client {
        let zp = appdata()?.join("EPIKPlayerBeta.zip");
        println!(" > $ Downloading EPIK17s Player...");
        filesdownloader(&cachelol(CZIPURL), &zp)?;
        unzip(&zp, &cdir()?)?;
        let _ = fs::remove_file(&zp);
        fuckoffprotocol();
        addprotocol()?;
    }

    if !sdir()?.exists() || !sdir()?.join("EPIKStudioBeta.exe").exists() {
        let zp = appdata()?.join("EPIKStudioBeta.zip");
        println!(" > $ Downloading EPIK17s Studio...");
        filesdownloader(&cachelol(SZIPURL), &zp)?;
        unzip(&zp, &sdir()?)?;
        let _ = fs::remove_file(&zp);
    }

    create_desktop_shortcut(&lpath()?, "EPIK17 Launcher")?;
    create_desktop_shortcut(&sdir()?.join("EPIKStudioBeta.exe"), "EPIK17 Studio")?;

    let old_lp = lp.with_extension("old");
    if old_lp.exists() {
        let _ = fs::remove_file(&old_lp);
    }

    Ok(())
}

fn epik17(url: &str) -> Result<(String, HashMap<String, String>)> {
    let u = url.replace("epik17:", "");
    let parts: Vec<&str> = u.split('+').collect();
    let mut map = HashMap::new();
    for p in &parts[1..] {
        if let Some(i) = p.find(':') {
            map.insert(p[..i].to_string(), decode(&p[i + 1..])?.to_string());
        }
    }
    Ok((parts[0].to_string(), map))
}

fn lclient(p: &HashMap<String, String>) -> Result<Child> {
    let ticket = p.get("ticket").cloned().unwrap_or_else(|| "whatyouwantthistobe".to_string());
    let gameid = p.get("gameid").cloned().unwrap_or_else(|| "1818".to_string());
    let base = "www.epik17.xyz";

    let (game_name, _) = get_game_details(&gameid);

    println!(" > $ Joining {}...", game_name);

    let game_exe = eepikexe()?;

    let game_child = Command::new(&game_exe)
        .arg("--play")
        .arg(format!("--authenticationUrl=https://{}/Login/Negotiate.ashx", base))
        .arg(format!("--authenticationTicket={}", ticket))
        .arg(format!(
            "--joinScriptUrl=https://{}/game/PlaceLauncher.ashx?placeId={}&t={}",
            base, gameid, ticket
        ))
        .spawn()?;

    Ok(game_child)
}

fn gamelmao() -> Result<()> {
    let _ = Command::new("cmd")
        .arg("/c")
        .arg("start")
        .arg("")
        .arg("https://www.epik17.xyz/games")
        .spawn()?;
    Ok(())
}

fn get_game_details(gameid: &str) -> (String, String) {
    let url = format!("https://epik17.xyz/v1/asset/details?id={}", gameid);
    if let Ok(response) = reqwest::blocking::get(&url) {
        if let Ok(details) = response.json::<AssetDetails>() {
            let creator_name = if details.creator.ischeckmarked {
                format!("{} ✅", details.creator.name)
            } else {
                details.creator.name
            };
            return (details.name, creator_name);
        }
    }
    (format!("Game ID: {}", gameid), "Unknown".to_string())
}

fn setup_discord_rpc(gameid: String) {
    std::thread::spawn(move || {
        let mut client = DiscordIpcClient::new(DSC_CLIENT_ID);
        
        let mut connected = false;
        for _ in 0..10 {
            if client.connect().is_ok() {
                connected = true;
                break;
            }
            std::thread::sleep(std::time::Duration::from_secs(2));
        }

        if !connected {
            return;
        }

        let (game_name, creator_name) = get_game_details(&gameid);
        
        let start_time = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs() as i64;

        let state_string = format!("By {}", creator_name);
        let game_url = format!("https://www.epik17.xyz/games/{}", gameid);
        let image_url = format!("https://www.epik17.xyz/Thumbs/Asset.ashx?width=420&height=420&assetId={}", gameid);
        
        let activity = activity::Activity::new()
            .details(&game_name)
            .state(&state_string)
            .timestamps(activity::Timestamps::new().start(start_time))
            .assets(activity::Assets::new().small_image(&image_url))
            .buttons(vec![
                activity::Button::new("See game page", &game_url)
            ]);

        let _ = client.set_activity(activity);

        loop {
            std::thread::sleep(std::time::Duration::from_secs(10));
        }
    });
}

fn run_app() -> Result<()> {
    execute!(stdout(), SetSize(55, 15), SetTitle("EPIK17s Launcher"))?;
    let banner = r#"
       EPIK17s  
          "#;

    println!("{}", banner);
    println!(" ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
    println!("                EPIK17s LAUNCHER v{}", VERSION);
    println!(" ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
    
    fs::create_dir_all(appdata()?)?;
    fs::create_dir_all(cdir()?)?;

    let lp = lpath()?;

    if !lp.exists() || env::current_exe()? != lp {
        if lp.exists() {
            let old_lp = lp.with_extension("old");
            let _ = fs::remove_file(&old_lp);
            let _ = fs::rename(&lp, &old_lp);
        }
        filesdownloader(&cachelol(EPIKLAUNCHER), &lp)?;
        addprotocol()?;
        gamelmao()?;
        Command::new(lp).spawn()?;
        std::process::exit(0);
    }

    updatechecker()?;
    addprotocol()?;

    if let Some(arg) = env::args().nth(1) {
        let (cmd, params) = epik17(&arg)?;
        if cmd == "play" {
            updatechecker()?;

            let game_child = lclient(&params)?;
            
            let gameid = params.get("gameid").cloned().unwrap_or_else(|| "Unknown".to_string());
            setup_discord_rpc(gameid);

            unsafe {
                let h = GetConsoleWindow();
                if !h.is_null() {
                    ShowWindow(h, 0); 
                }
            }

            let game_arc = Arc::new(Mutex::new(game_child));
            let game_clone = Arc::clone(&game_arc);

            ctrlc::set_handler(move || {
                let _ = game_clone.lock().unwrap().kill();
                std::process::exit(0);
            })?;

            game_arc.lock().unwrap().wait()?;
        }
    }

    Ok(())
}

fn main() {
    if let Err(err) = run_app() {
        eprintln!("\n > $ [CRITICAL ERROR!] {}", err);
        
        for cause in err.chain().skip(1) {
            eprintln!("    Caused by: {}", cause);
        }
        
        println!("\nPress ENTER to close this window.");
        let mut dump = String::new();
        let _ = std::io::stdin().read_line(&mut dump);
    }
}
// epik17s is modifed epik17!
// it's a modifed epik17 revival made by guutut himself.
// if you want to contact me in discord heres my username: nzaonpekora
// have fun on epik17s!
