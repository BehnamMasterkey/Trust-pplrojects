Main.unityGameManager.csPuzzlePiece.csAssets/
 ├── Scenes/
 │    └── Main.unity
 ├── Scripts/
 │    ├── GameManager.cs
 │    ├── PuzzlePiece.cs
 │    ├── AudioManager.cs
 │    └── UIManager.cs
 ├── Prefabs/
 │    └── PuzzlePiecePrefab.prefab
 ├── Resources/
 │    ├── Images/
 │    │     ├── Level1_Shahr_e_Sukhteh.png
 │    │     ├── Level2_Persepolis.png
 │    │     ├── Level3_Dariush.png
 │    │     ├── Level4_Cyrus.png
 │    │     └── Level5_CyrusCylinder.png
 │    └── Audio/
 │          ├── bg1_Sukhteh.mp3
 │          ├── bg2_Persepolis.mp3
 │          ├── bg3_Dariush.mp3
 │          ├── bg4_Cyrus.mp3
 │          ├── bg5_Cylinder.mp3
 │          └── sfx_daf.wav
 └── UI/
      └── Canvas (در صحنه)GameManager.csPuzzlePiece.csUIManager.csAudioManager.csPuzzlePiecePrefab.prefabLevel1_Shahr_e_Sukhteh.pngLevel3_Dariush.pngLevel2_Persepolis.pngLevel4_Cyrus.pngLevel5_CyrusCylinder.pngusing System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class GameManager : MonoBehaviour
{
    public static GameManager Instance;

    [Header("تنظیمات عمومی")]
    public RectTransform boardContainer; // محل قرارگیری قطعات پازل در Canvas
    public GameObject piecePrefab;       // Prefab قطعه پازل
    public Text levelText;               // متن شماره مرحله
    public Button nextLevelButton;       // دکمه مرحله بعد
    public Button restartButton;         // دکمه تکرار مرحله

    [Header("نام تصاویر مراحل (مسیر در Resources/Images)")]
    public string[] levelImageNames = new string[] {
        "Images/Level1_Shahr_e_Sukhteh",
        "Images/Level2_Persepolis",
        "Images/Level3_Dariush",
        "Images/Level4_Cyrus",
        "Images/Level5_CyrusCylinder"
    };

    [Header("موسیقی پس‌زمینه برای هر مرحله (در Resources/Audio)")]
    public string[] levelMusicNames = new string[] {
        "Audio/bg1_Sukhteh",
        "Audio/bg2_Persepolis",
        "Audio/bg3_Dariush",
        "Audio/bg4_Cyrus",
        "Audio/bg5_Cylinder"
    };

    // تعداد قطعات هر مرحله (هر مرحله دو برابر مرحله قبل)
    int[] pieceCounts = new int[] { 16, 32, 64, 128, 256 };

    int currentLevel = 0;
    List<GameObject> spawnedPieces = new List<GameObject>();
    bool levelCompleted = false;

    void Awake()
    {
        if (Instance == null) Instance = this;
        else Destroy(gameObject);
    }

    void Start()
    {
        nextLevelButton.onClick.AddListener(NextLevel);
        restartButton.onClick.AddListener(RestartLevel);
        LoadLevel(0);
    }

    /// <summary>
    /// بارگذاری مرحله جدید
    /// </summary>
    public void LoadLevel(int levelIndex)
    {
        ClearBoard();
        currentLevel = Mathf.Clamp(levelIndex, 0, pieceCounts.Length - 1);
        levelText.text = $"مرحله {currentLevel + 1}";
        levelCompleted = false;

        // پخش موسیقی مناسب مرحله
        AudioManager.Instance.PlayBackground(levelMusicNames[currentLevel]);

        // لود تصویر مرحله از Resources
        Sprite levelSprite = Resources.Load<Sprite>(levelImageNames[currentLevel]);
        if (levelSprite == null)
        {
            Debug.LogError("تصویر مرحله یافت نشد: " + levelImageNames[currentLevel]);
            return;
        }

        StartCoroutine(SpawnPieces(levelSprite, pieceCounts[currentLevel]));
    }

    IEnumerator SpawnPieces(Sprite sprite, int pieces)
    {
        int cols = Mathf.RoundToInt(Mathf.Sqrt(pieces));
        int rows = Mathf.CeilToInt((float)pieces / cols);

        var grid = boardContainer.GetComponent<GridLayoutGroup>();
        if (grid == null) grid = boardContainer.gameObject.AddComponent<GridLayoutGroup>();

        float availableW = boardContainer.rect.width;
        float availableH = boardContainer.rect.height;
        float cell = Mathf.Min(availableW / cols, availableH / rows);
        grid.cellSize = new Vector2(cell - 2, cell - 2);
        grid.constraint = GridLayoutGroup.Constraint.FixedColumnCount;
        grid.constraintCount = cols;

        // تکه‌تکه کردن تصویر به قطعات کوچک‌تر
        Texture2D tex = sprite.texture;
        int texW = tex.width;
        int texH = tex.height;

        List<Sprite> slices = new List<Sprite>();
        int counter = 0;
        for (int y = 0; y < rows; y++)
        {
            for (int x = 0; x < cols; x++)
            {
                if (counter >= pieces) break;
                Rect rect = new Rect(x * texW / cols, y * texH / rows, texW / cols, texH / rows);
                Sprite s = Sprite.Create(tex, rect, new Vector2(0.5f, 0.5f), sprite.pixelsPerUnit);
                slices.Add(s);
                counter++;
            }
        }

        // ساخت قطعات پازل
        foreach (var spr in slices)
        {
            GameObject piece = Instantiate(piecePrefab, boardContainer);
            piece.GetComponent<PuzzlePiece>().Init(spr);
            spawnedPieces.Add(piece);
            yield return null;
        }

        ShufflePieces();
    }

    void ShufflePieces()
    {
        for (int i = 0; i < spawnedPieces.Count; i++)
        {
            int rand = Random.Range(0, spawnedPieces.Count);
            var tmp = spawnedPieces[i].transform.GetSiblingIndex();
            spawnedPieces[i].transform.SetSiblingIndex(rand);
        }
    }

    public void CheckPuzzleCompletion()
    {
        // بررسی اینکه همه قطعات در مکان صحیح هستند یا نه
        foreach (var piece in spawnedPieces)
        {
            if (!piece.GetComponent<PuzzlePiece>().IsCorrect())
                return;
        }

        if (!levelCompleted)
        {
            levelCompleted = true;
            StartCoroutine(PlayCompletionEffects());
        }
    }

    IEnumerator PlayCompletionEffects()
    {
        // صدای دف هنگام اتمام مرحله
        AudioManager.Instance.PlaySFX("Audio/sfx_daf");

        // افکت محو شدن تدریجی و بزرگ شدن تصویر نهایی
        yield return new WaitForSeconds(0.3f);
        boardContainer.gameObject.AddComponent<Animator>().enabled = true;

        // نمایش دکمه مرحله بعد
        yield return new WaitForSeconds(2f);
        nextLevelButton.gameObject.SetActive(true);
    }

    void ClearBoard()
    {
        foreach (var g in spawnedPieces)
            Destroy(g);
        spawnedPieces.Clear();
        nextLevelButton.gameObject.SetActive(false);
    }

    public void NextLevel()
    {
        if (currentLevel + 1 < pieceCounts.Length)
            LoadLevel(currentLevel + 1);
        else
            LoadLevel(0);
    }

    public void RestartLevel()
    {
        LoadLevel(currentLevel);
    }
}sfx_daf.wav# Trust-pplrojects
Phenomenal crashing 
