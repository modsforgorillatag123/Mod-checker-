using MelonLoader;
using UnityEngine;
using System;
using System.Linq;
using System.Reflection;
using System.Collections.Generic;

[assembly: MelonInfo(typeof(ModChecker.ModCheckerMod), "Gorilla Tag ModChecker", "1.0.0", "modsforgorillatag123")]
[assembly: MelonGame("Gorilla Tag", "Gorilla Tag")]

namespace ModChecker
{
    public class ModCheckerMod : MelonMod
    {
        private Rect windowRect = new Rect(20, 20, 560, 420);
        private bool showWindow = true;
        private Vector2 scroll;
        private List<ModEntry> mods = new List<ModEntry>();
        private string exportPath;
        private HashSet<string> knownBadKeywords = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
        {
            "cheat","aim","bot","noclip","teleport","autoswing","wallhack","esp","speed","fly","nocooldown","silent"
        };

        public override void OnInitializeMelon()
        {
            MelonLogger.Msg("ModChecker initializing");
            exportPath = System.IO.Path.Combine(Application.persistentDataPath, "mod_report.json");
            Refresh();

            // Create a GameObject to ensure a persistent OnGUI route
            var go = new GameObject("ModCheckerGUI");
            UnityEngine.Object.DontDestroyOnLoad(go);
            go.AddComponent<GuiBehaviour>().Init(this);
        }

        public void Refresh()
        {
            mods.Clear();
            var assemblies = AppDomain.CurrentDomain.GetAssemblies().OrderBy(a => a.GetName().Name);
            foreach (var a in assemblies)
            {
                try
                {
                    var name = a.GetName().Name ?? "unknown";
                    var version = a.GetName().Version?.ToString() ?? "n/a";
                    var loc = a.IsDynamic ? "dynamic" : (string.IsNullOrEmpty(a.Location) ? "unknown" : a.Location);
                    bool likely = IsLikelyMod(a);
                    bool flaggedBad = knownBadKeywords.Any(k => name.IndexOf(k, StringComparison.OrdinalIgnoreCase) >= 0);
                    var entry = new ModEntry { AssemblyName = name, Version = version, Location = loc, IsLikelyMod = likely, IsFlagged = flaggedBad };
                    mods.Add(entry);
                }
                catch
                {
                    // ignore problematic assemblies
                }
            }
        }

        bool IsLikelyMod(Assembly a)
        {
            var n = a.GetName().Name ?? "";
            if (n.IndexOf("mod", StringComparison.OrdinalIgnoreCase) >= 0 ||
                n.IndexOf("melons", StringComparison.OrdinalIgnoreCase) >= 0 ||
                n.IndexOf("bepin", StringComparison.OrdinalIgnoreCase) >= 0 ||
                n.IndexOf("gorilla", StringComparison.OrdinalIgnoreCase) >= 0)
                return true;

            try
            {
                foreach (var t in a.GetTypes())
                {
                    var fn = t.FullName ?? "";
                    if (fn.StartsWith("MelonLoader.") || fn.StartsWith("BepInEx.") || (fn.Contains("Gorilla") && fn.Contains("Mod")))
                        return true;
                }
            }
            catch
            {
                // GetTypes can throw for some assemblies, ignore
            }

            return false;
        }

        public void DrawWindow()
        {
            if (!showWindow) return;
            windowRect = GUI.Window(1337, windowRect, WindowFunc, "Mod Checker (Steam PC)");
        }

        void WindowFunc(int id)
        {
            GUILayout.BeginVertical();

            GUILayout.BeginHorizontal();
            if (GUILayout.Button("Refresh", GUILayout.Width(100))) Refresh();
            if (GUILayout.Button("Export JSON", GUILayout.Width(120))) Export();
            if (GUILayout.Button(showWindow ? "Hide" : "Show", GUILayout.Width(80))) showWindow = !showWindow;
            GUILayout.EndHorizontal();

            GUILayout.Space(6);

            scroll = GUILayout.BeginScrollView(scroll, GUILayout.Height(320));
            foreach (var m in mods)
            {
                GUILayout.BeginHorizontal();
                GUILayout.Label(m.AssemblyName, GUILayout.Width(260));
                GUILayout.Label(m.Version, GUILayout.Width(80));
                GUILayout.Label(m.IsFlagged ? "<color=red>FLAGGED</color>" : (m.IsLikelyMod ? "<color=yellow>LIKELY</color>" : "<color=green>OK</color>"));
                GUILayout.EndHorizontal();
                GUILayout.Label("  " + m.Location);
                GUILayout.Space(6);
            }
            GUILayout.EndScrollView();

            GUILayout.EndVertical();
            GUI.DragWindow();
        }

        void Export()
        {
            var report = new ModReport { Timestamp = DateTime.UtcNow.ToString("o"), Mods = mods.ToArray() };
            var json = JsonUtility.ToJson(report, true);
            try
            {
                System.IO.File.WriteAllText(exportPath, json);
                MelonLogger.Msg($"Exported mod report to: {exportPath}");
            }
            catch (Exception e)
            {
                MelonLogger.Warning($"Failed to export: {e}");
            }
        }

        [Serializable]
        public class ModEntry
        {
            public string AssemblyName;
            public string Version;
            public string Location;
            public bool IsLikelyMod;
            public bool IsFlagged;
        }

        [Serializable]
        public class ModReport
        {
            public string Timestamp;
            public ModEntry[] Mods;
        }
    }

    // Small MonoBehaviour to route OnGUI reliably across scenes
    public class GuiBehaviour : MonoBehaviour
    {
        private ModCheckerMod parent;
        public void Init(ModCheckerMod p) { parent = p; }
        void OnGUI() { parent?.DrawWindow(); }
    }
}
