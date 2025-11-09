#!/usr/bin/env python3
"""
Quick GitHub Profile Scanner
Run this script to automatically scan your repositories and update your profile
"""

import json
import subprocess
import sys
import os
import glob
from datetime import datetime

PREFERENCES_FILE = "profile_preferences.json"
PROFILE_OPTION_KEYS = (
    "about",
    "tagline",
    "focus",
    "learning",
    "open_to",
    "contact_email",
    "linkedin",
    "portfolio",
    "twitter",
    "highlighted_repos",
)


def install_requirements():
    """Install required packages"""
    required_packages = ['requests']

    for package in required_packages:
        try:
            __import__(package)
        except ImportError:
            print(f"Installing {package}...")
            subprocess.check_call([sys.executable, "-m", "pip", "install", package])


def load_preferences():
    """Load stored profile preferences if available"""
    if os.path.exists(PREFERENCES_FILE):
        try:
            with open(PREFERENCES_FILE, "r", encoding="utf-8") as handle:
                data = json.load(handle)
                highlighted = data.get("highlighted_repos")
                if isinstance(highlighted, str):
                    data["highlighted_repos"] = [item.strip() for item in highlighted.split(",") if item.strip()]
                return data
        except json.JSONDecodeError:
            print("⚠️ Could not read profile preferences file. Creating a fresh one.")
    return {}


def save_preferences(preferences):
    """Persist profile preferences for future runs"""
    with open(PREFERENCES_FILE, "w", encoding="utf-8") as handle:
        json.dump(preferences, handle, indent=2)


def prompt_with_default(message, default_value="", required=False):
    """Prompt the user for input while supporting default values"""
    while True:
        suffix = f" [{default_value}]" if default_value else ""
        response = input(f"{message}{suffix}: ").strip()

        if response:
            return response
        if default_value:
            return default_value
        if not required:
            return ""
        print("Please provide a value for this prompt.")


def parse_cli_arguments():
    """Extract username argument and flags from the CLI"""
    username_arg = None
    skip_prompts = False

    for arg in sys.argv[1:]:
        if arg in {"--no-input", "--no-prompt"}:
            skip_prompts = True
        elif not arg.startswith("-") and username_arg is None:
            username_arg = arg

    return username_arg, skip_prompts


def gather_user_preferences(existing_preferences, username_arg=None, skip_prompts=False):
    """Collect or reuse profile preferences from the user"""
    preferences = existing_preferences.copy()

    current_username = username_arg or preferences.get("username", "")

    if skip_prompts and not current_username:
        raise ValueError("GitHub username is required. Provide it as an argument or run without --no-input.")

    if not skip_prompts or not current_username:
        current_username = prompt_with_default(
            "Enter your GitHub username",
            current_username,
            required=True,
        )

    preferences["username"] = current_username

    if skip_prompts:
        return preferences

    print("\nLet's capture a few optional details to personalize your profile (press Enter to skip).")

    optional_fields = (
        ("about", "Custom 'About' paragraph to override your GitHub bio"),
        ("tagline", "Short personal tagline (e.g. Software Developer & AI Enthusiast)"),
        ("focus", "What are you building right now?"),
        ("learning", "What are you currently learning?"),
        ("open_to", "Opportunities you're open to"),
        ("contact_email", "Preferred contact email"),
        ("linkedin", "LinkedIn URL"),
        ("portfolio", "Portfolio URL"),
        ("twitter", "Twitter/X handle or URL"),
        (
            "highlighted_repos",
            "Comma-separated repository names you want always highlighted (e.g. repo-one, repo-two)",
        ),
    )

    for key, prompt_text in optional_fields:
        default_val = preferences.get(key, "")
        if key == "highlighted_repos" and isinstance(default_val, list):
            default_val = ", ".join(default_val)
        value = prompt_with_default(prompt_text, default_val)
        if key == "highlighted_repos":
            if value:
                preferences[key] = [item.strip() for item in value.split(",") if item.strip()]
            else:
                preferences[key] = []
        else:
            preferences[key] = value

    return preferences


def run_analysis(username, profile_options):
    """Run the GitHub profile analysis"""
    try:
        # Import after ensuring requirements are installed
        from github_profile_analyzer import GitHubProfileAnalyzer

        print(f"🔍 Scanning GitHub repositories for {username}...")
        analyzer = GitHubProfileAnalyzer(username)

        # Generate updated README
        readme_content = analyzer.generate_readme_content(profile_options)

        if readme_content:
            # Backup existing README if it exists
            if os.path.exists("README.md"):
                timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
                backup_name = f"README_backup_{timestamp}.md"
                os.rename("README.md", backup_name)
                print(f"📄 Backed up existing README.md as {backup_name}")

            # Save new README
            with open("README.md", "w", encoding="utf-8") as handle:
                handle.write(readme_content)

            # Save detailed analysis
            analyzer.save_profile_data("profile_data.json", extras={"preferences": profile_options})

            print("✅ Profile scan completed successfully!")
            print("📄 Updated README.md with latest repository information")
            print("📊 Saved detailed analysis to profile_data.json")

            # Display summary
            stats = analyzer.generate_stats_section()
            languages = analyzer.analyze_languages()

            print("\n📈 Profile Summary:")
            print(f"   Total Repositories: {stats['total_repos']}")
            print(f"   Total Stars: {stats['total_stars']}")
            print(f"   Programming Languages: {len(languages)}")
            print(f"   Top Language: {list(languages.keys())[0] if languages else 'None'}")
            print(f"   Followers: {stats['followers']}")

        else:
            print("❌ Failed to scan repositories. Please check your internet connection and username.")

    except Exception as exc:
        print(f"❌ Error during analysis: {exc}")


def cleanup_old_backups():
    """Remove old backup files, keeping only the 3 most recent"""
    backup_files = glob.glob("README_backup_*.md")
    if len(backup_files) > 3:
        # Sort by creation time and remove oldest
        backup_files.sort(key=os.path.getctime)
        for old_backup in backup_files[:-3]:
            os.remove(old_backup)
            print(f"🗑️ Removed old backup: {old_backup}")


def main():
    print("🚀 GitHub Profile Auto-Scanner")
    print("=" * 40)

    username_arg, skip_prompts = parse_cli_arguments()
    preferences = load_preferences()

    try:
        preferences = gather_user_preferences(preferences, username_arg, skip_prompts)
    except ValueError as error:
        print(f"❌ {error}")
        return

    save_preferences(preferences)

    username = preferences["username"]
    profile_options = {}
    for key, value in preferences.items():
        if key not in PROFILE_OPTION_KEYS:
            continue
        if key == "highlighted_repos":
            if isinstance(value, list):
                cleaned = [item.strip() for item in value if item.strip()]
            else:
                cleaned = [item.strip() for item in str(value).split(",") if item.strip()]
            if cleaned:
                profile_options[key] = cleaned
            continue

        if isinstance(value, str):
            stripped = value.strip()
            if stripped:
                profile_options[key] = stripped
        elif value:
            profile_options[key] = value

    # Install requirements
    install_requirements()

    # Clean up old backups
    cleanup_old_backups()

    # Run analysis
    run_analysis(username, profile_options)

    print("\n🎉 Profile update complete!")
    print("Your README.md has been updated with the latest information from your repositories.")


if __name__ == "__main__":
    main()
