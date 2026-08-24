Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# ==========================================
# SNAKE - Resizable Windows PowerShell Game
# ==========================================

$gridWidth = 30
$gridHeight = 20

$snake = @()
$food = $null

$direction = "Right"
$nextDirection = "Right"

$score = 0
$gameOver = $false

# -------------------------
# Window
# -------------------------

$form = New-Object System.Windows.Forms.Form
$form.Text = "Snake"
$form.ClientSize = New-Object System.Drawing.Size(1000, 700)
$form.MinimumSize = New-Object System.Drawing.Size(600, 450)
$form.StartPosition = "CenterScreen"
$form.BackColor = [System.Drawing.Color]::Black
$form.KeyPreview = $true

# -------------------------
# Game panel
# -------------------------

$panel = New-Object System.Windows.Forms.Panel
$panel.Dock = "Fill"
$panel.BackColor = [System.Drawing.Color]::Black
$form.Controls.Add($panel)

# -------------------------
# Bottom controls
# -------------------------

$bottomPanel = New-Object System.Windows.Forms.Panel
$bottomPanel.Dock = "Bottom"
$bottomPanel.Height = 55
$bottomPanel.BackColor = [System.Drawing.Color]::FromArgb(25,25,25)
$form.Controls.Add($bottomPanel)

# Score
$scoreLabel = New-Object System.Windows.Forms.Label
$scoreLabel.Text = "Score: 0"
$scoreLabel.ForeColor = [System.Drawing.Color]::White
$scoreLabel.BackColor = [System.Drawing.Color]::Transparent
$scoreLabel.Font = New-Object System.Drawing.Font(
    "Arial",
    12,
    [System.Drawing.FontStyle]::Bold
)
$scoreLabel.AutoSize = $true
$scoreLabel.Location = New-Object System.Drawing.Point(15, 16)
$bottomPanel.Controls.Add($scoreLabel)

# Speed label
$speedLabel = New-Object System.Windows.Forms.Label
$speedLabel.Text = "Speed:"
$speedLabel.ForeColor = [System.Drawing.Color]::White
$speedLabel.BackColor = [System.Drawing.Color]::Transparent
$speedLabel.Font = New-Object System.Drawing.Font("Arial", 10)
$speedLabel.AutoSize = $true
$speedLabel.Location = New-Object System.Drawing.Point(130, 18)
$bottomPanel.Controls.Add($speedLabel)

# Speed slider
$speedSlider = New-Object System.Windows.Forms.TrackBar
$speedSlider.Minimum = 1
$speedSlider.Maximum = 10
$speedSlider.Value = 5
$speedSlider.TickFrequency = 1
$speedSlider.SmallChange = 1
$speedSlider.LargeChange = 1
$speedSlider.Width = 190
$speedSlider.Height = 40
$speedSlider.Location = New-Object System.Drawing.Point(180, 7)
$bottomPanel.Controls.Add($speedSlider)

# Speed number
$speedValueLabel = New-Object System.Windows.Forms.Label
$speedValueLabel.Text = "5"
$speedValueLabel.ForeColor = [System.Drawing.Color]::White
$speedValueLabel.BackColor = [System.Drawing.Color]::Transparent
$speedValueLabel.Font = New-Object System.Drawing.Font(
    "Arial",
    10,
    [System.Drawing.FontStyle]::Bold
)
$speedValueLabel.AutoSize = $true
$speedValueLabel.Location = New-Object System.Drawing.Point(375, 18)
$bottomPanel.Controls.Add($speedValueLabel)

# Restart button
$restartButton = New-Object System.Windows.Forms.Button
$restartButton.Text = "Restart"
$restartButton.Size = New-Object System.Drawing.Size(90, 30)
$restartButton.Anchor = "Top,Right"
$restartButton.Location = New-Object System.Drawing.Point(890, 12)
$restartButton.Visible = $false
$bottomPanel.Controls.Add($restartButton)

# -------------------------
# Board sizing
# -------------------------

function Get-BoardSettings {

    $availableWidth = $panel.ClientSize.Width
    $availableHeight = $panel.ClientSize.Height

    $cellWidth = [Math]::Floor($availableWidth / $gridWidth)
    $cellHeight = [Math]::Floor($availableHeight / $gridHeight)

    $script:drawCell = [Math]::Max(
        5,
        [Math]::Min($cellWidth, $cellHeight)
    )

    $script:boardWidth = $drawCell * $gridWidth
    $script:boardHeight = $drawCell * $gridHeight

    $script:boardX = [Math]::Floor(
        ($availableWidth - $boardWidth) / 2
    )

    $script:boardY = [Math]::Floor(
        ($availableHeight - $boardHeight) / 2
    )
}

# -------------------------
# Create food
# -------------------------

function New-Food {

    do {

        $x = Get-Random -Minimum 0 -Maximum $gridWidth
        $y = Get-Random -Minimum 0 -Maximum $gridHeight

        $occupied = $false

        foreach ($part in $snake) {

            if ($part.X -eq $x -and $part.Y -eq $y) {
                $occupied = $true
                break
            }
        }

    } while ($occupied)

    return [PSCustomObject]@{
        X = $x
        Y = $y
    }
}

# -------------------------
# Start / Restart
# -------------------------

function Start-Game {

    $script:snake = @(
        [PSCustomObject]@{ X = 15; Y = 10 }
        [PSCustomObject]@{ X = 14; Y = 10 }
        [PSCustomObject]@{ X = 13; Y = 10 }
    )

    $script:direction = "Right"
    $script:nextDirection = "Right"

    $script:score = 0
    $script:gameOver = $false

    $scoreLabel.Text = "Score: 0"
    $restartButton.Visible = $false

    $script:food = New-Food

    # Apply current speed
    $speed = $speedSlider.Value
    $timer.Interval = 210 - ($speed * 20)

    Get-BoardSettings

    $timer.Start()

    $panel.Invalidate()
}

# -------------------------
# Game timer
# -------------------------

$timer = New-Object System.Windows.Forms.Timer

# Default speed
$timer.Interval = 110

$timer.Add_Tick({

    if ($gameOver) {
        return
    }

    $script:direction = $nextDirection

    $head = $snake[0]

    # Calculate new head
    switch ($direction) {

        "Up" {
            $newHead = [PSCustomObject]@{
                X = $head.X
                Y = $head.Y - 1
            }
        }

        "Down" {
            $newHead = [PSCustomObject]@{
                X = $head.X
                Y = $head.Y + 1
            }
        }

        "Left" {
            $newHead = [PSCustomObject]@{
                X = $head.X - 1
                Y = $head.Y
            }
        }

        "Right" {
            $newHead = [PSCustomObject]@{
                X = $head.X + 1
                Y = $head.Y
            }
        }
    }

    # -------------------------
    # Wall collision
    # -------------------------

    if (
        $newHead.X -lt 0 -or
        $newHead.X -ge $gridWidth -or
        $newHead.Y -lt 0 -or
        $newHead.Y -ge $gridHeight
    ) {

        $script:gameOver = $true
        $timer.Stop()
        $restartButton.Visible = $true
        $panel.Invalidate()

        return
    }

    # -------------------------
    # Snake collision
    # -------------------------

    foreach ($part in $snake) {

        if (
            $newHead.X -eq $part.X -and
            $newHead.Y -eq $part.Y
        ) {

            $script:gameOver = $true
            $timer.Stop()
            $restartButton.Visible = $true
            $panel.Invalidate()

            return
        }
    }

    # -------------------------
    # Add new head
    # -------------------------

    $script:snake = @($newHead) + $snake

    # -------------------------
    # Eat food
    # -------------------------

    if (
        $newHead.X -eq $food.X -and
        $newHead.Y -eq $food.Y
    ) {

        $script:score++

        $scoreLabel.Text = "Score: $score"

        $script:food = New-Food
    }
    else {

        # Remove tail
        if ($snake.Count -gt 1) {
            $script:snake = $snake[0..($snake.Count - 2)]
        }
    }

    $panel.Invalidate()
})

# -------------------------
# Speed slider
# -------------------------

$speedSlider.Add_ValueChanged({

    $speed = $speedSlider.Value

    # 1 = slowest
    # 10 = fastest
    $timer.Interval = 210 - ($speed * 20)

    $speedValueLabel.Text = "$speed"
})

# Prevent arrow keys from changing the slider
$speedSlider.Add_KeyDown({

    param($sender, $e)

    if (
        $e.KeyCode -eq [System.Windows.Forms.Keys]::Left -or
        $e.KeyCode -eq [System.Windows.Forms.Keys]::Right -or
        $e.KeyCode -eq [System.Windows.Forms.Keys]::Up -or
        $e.KeyCode -eq [System.Windows.Forms.Keys]::Down
    ) {

        $e.SuppressKeyPress = $true
        $e.Handled = $true
    }
})

# -------------------------
# Drawing
# -------------------------

$panel.Add_Paint({

    param($sender, $e)

    Get-BoardSettings

    $g = $e.Graphics

    # Background
    $g.Clear([System.Drawing.Color]::Black)

    # -------------------------
    # Board
    # -------------------------

    $boardBrush = New-Object System.Drawing.SolidBrush(
        [System.Drawing.Color]::FromArgb(12,12,12)
    )

    $g.FillRectangle(
        $boardBrush,
        $boardX,
        $boardY,
        $boardWidth,
        $boardHeight
    )

    $boardBrush.Dispose()

    # -------------------------
    # Grid
    # -------------------------

    $gridPen = New-Object System.Drawing.Pen(
        [System.Drawing.Color]::FromArgb(30,30,30)
    )

    for ($x = 0; $x -le $gridWidth; $x++) {

        $px = $boardX + ($x * $drawCell)

        $g.DrawLine(
            $gridPen,
            $px,
            $boardY,
            $px,
            $boardY + $boardHeight
        )
    }

    for ($y = 0; $y -le $gridHeight; $y++) {

        $py = $boardY + ($y * $drawCell)

        $g.DrawLine(
            $gridPen,
            $boardX,
            $py,
            $boardX + $boardWidth,
            $py
        )
    }

    $gridPen.Dispose()

    # -------------------------
    # Food
    # -------------------------

    if ($food) {

        $foodBrush = New-Object System.Drawing.SolidBrush(
            [System.Drawing.Color]::Red
        )

        $fx = $boardX + ($food.X * $drawCell)
        $fy = $boardY + ($food.Y * $drawCell)

        $g.FillEllipse(
            $foodBrush,
            $fx + 3,
            $fy + 3,
            $drawCell - 6,
            $drawCell - 6
        )

        $foodBrush.Dispose()
    }

    # -------------------------
    # Snake
    # -------------------------

    for ($i = 0; $i -lt $snake.Count; $i++) {

        $part = $snake[$i]

        if ($i -eq 0) {
            $color = [System.Drawing.Color]::LimeGreen
        }
        else {
            $color = [System.Drawing.Color]::Green
        }

        $brush = New-Object System.Drawing.SolidBrush($color)

        $sx = $boardX + ($part.X * $drawCell)
        $sy = $boardY + ($part.Y * $drawCell)

        $g.FillRectangle(
            $brush,
            $sx + 2,
            $sy + 2,
            $drawCell - 4,
            $drawCell - 4
        )

        $brush.Dispose()
    }

    # -------------------------
    # Game Over
    # -------------------------

    if ($gameOver) {

        $overlay = New-Object System.Drawing.SolidBrush(
            [System.Drawing.Color]::FromArgb(180,0,0,0)
        )

        $g.FillRectangle(
            $overlay,
            $boardX,
            $boardY,
            $boardWidth,
            $boardHeight
        )

        $overlay.Dispose()

        $fontSize = [Math]::Max(
            24,
            [Math]::Floor($drawCell * 1.5)
        )

        $font = New-Object System.Drawing.Font(
            "Arial",
            $fontSize,
            [System.Drawing.FontStyle]::Bold
        )

        $textBrush = New-Object System.Drawing.SolidBrush(
            [System.Drawing.Color]::White
        )

        $text = "GAME OVER"

        $textSize = $g.MeasureString(
            $text,
            $font
        )

        $tx = $boardX +
            (($boardWidth - $textSize.Width) / 2)

        $ty = $boardY +
            (($boardHeight - $textSize.Height) / 2)

        $g.DrawString(
            $text,
            $font,
            $textBrush,
            $tx,
            $ty
        )

        $textBrush.Dispose()
        $font.Dispose()
    }
})

# -------------------------
# Keyboard controls
# -------------------------

$form.Add_KeyDown({

    param($sender, $e)

    switch ($e.KeyCode) {

        "Up" {

            if ($direction -ne "Down") {
                $script:nextDirection = "Up"
            }
        }

        "Down" {

            if ($direction -ne "Up") {
                $script:nextDirection = "Down"
            }
        }

        "Left" {

            if ($direction -ne "Right") {
                $script:nextDirection = "Left"
            }
        }

        "Right" {

            if ($direction -ne "Left") {
                $script:nextDirection = "Right"
            }
        }

        "Escape" {
            $form.Close()
        }
    }
})

# -------------------------
# Restart button
# -------------------------

$restartButton.Add_Click({

    Start-Game

    $form.Focus()
})

# -------------------------
# Resize handling
# -------------------------

$form.Add_Resize({

    if ($panel) {

        Get-BoardSettings

        $panel.Invalidate()
    }

    if ($restartButton -and $bottomPanel) {

        $restartButton.Location = New-Object System.Drawing.Point(
            ($bottomPanel.ClientSize.Width - 100),
            12
        )
    }
})

# -------------------------
# Start
# -------------------------

Get-BoardSettings
Start-Game

[void]$form.ShowDialog()



