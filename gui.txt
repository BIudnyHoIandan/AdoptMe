local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

-- Assuming the ClientDataModule is accessible on the client as used in the original script
local ClientDataModule = require(ReplicatedStorage.ClientModules.Core.ClientData)

-- Remote Call Definitions
-- IMPORTANT: These wait for the server components to exist before proceeding.
local API = ReplicatedStorage:WaitForChild("API")
local AddPetRemote = API:WaitForChild("IdleProgressionAPI.AddPet")
-- Remote for removing pets
local RemovePetRemote = API:WaitForChild("IdleProgressionAPI.RemovePet")
-- Remote for claiming progression/XP
local CommitProgressionRemote = API:WaitForChild("IdleProgressionAPI.CommitAllProgression")

-- Configuration
local TARGET_SPECIES_IDS = {
    -- Add the Species IDs you want to highlight or filter here:
    -- "rare_kitty",
    -- "legendary_phoenix",
}

local function waitForData()
    -- This function remains, but we will visually show the loading state in the UI
    local data = ClientDataModule.get_data()
    while not data do
        task.wait(0.5)
        data = ClientDataModule.get_data()
    end
    return data
end

-- ====================================================================
-- || UTILITY FUNCTIONS ||
-- ====================================================================

-- Function to simulate a button click by firing all connected events
local function triggerButton(button)
    if button and button:IsA("GuiButton") then
        -- This logic ensures any script connected to these events is triggered
        local success, err = pcall(function()
            for _, connection in pairs(getconnections(button.MouseButton1Down)) do
                connection:Fire()
            end
            for _, connection in pairs(getconnections(button.MouseButton1Click)) do
                connection:Fire()
            end
            for _, connection in pairs(getconnections(button.MouseButton1Up)) do
                connection:Fire()
            end
        end)
        
        if not success then
            warn("Failed to trigger button connections: " .. tostring(err))
        end
    end
end

-- NEW: Function to start the continuous auto-close loop for the Pet Level Up app
local function startAutoCloseLoop()
    task.spawn(function()
        local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
        
        while true do
            -- Safely navigate the deep path
            local PetLevelUpApp = PlayerGui:FindFirstChild("PetLevelUpApp")
            
            if PetLevelUpApp then
                local Frame = PetLevelUpApp:FindFirstChild("Frame")
                
                -- Check for the visibility of the whole app
                if Frame and PetLevelUpApp.Enabled and PetLevelUpApp.Frame.Visible then
                    
                    local Container = Frame:FindFirstChild("Container")
                    local FooterButton = Container and Container:FindFirstChild("FooterButton")
                    local DoneButton = FooterButton and FooterButton:FindFirstChild("DoneButton")
                    local targetButton = DoneButton and DoneButton:FindFirstChild("DepthButton")
                    
                    if targetButton and targetButton:IsA("GuiButton") and targetButton.Visible then
                        print("Auto-closing Pet Level Up notification...")
                        triggerButton(targetButton)
                        -- Wait a moment after clicking to prevent repeated fast clicks
                        task.wait(1) 
                    end
                end
            end

            task.wait(0.2) -- Check every 0.2 seconds for responsiveness
        end
    end)
    print("Auto-Close loop for PetLevelUpApp started.")
end

-- Function to start the continuous XP claim loop
local function startXpClaimLoop()
    task.spawn(function()
        -- Loop indefinitely in a separate thread
        while true do
            -- Fire the remote with no arguments to commit all progression
            CommitProgressionRemote:FireServer()
            -- Wait for a short interval (e.g., 5 seconds) to prevent throttling/spamming
            task.wait(5) 
        end
    end)
    print("XP Claim Loop started (CommitAllProgression every 5 seconds).")
end

-- ====================================================================
-- || UI CREATION FUNCTIONS ||
-- ====================================================================

local function createUI(screenGui)
    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "PetSelectorFrame"
    MainFrame.Size = UDim2.new(0.4, 0, 0.7, 0) -- 40% width, 70% height
    MainFrame.Position = UDim2.fromScale(0.5, 0.5) -- Simplified centering using Scale
    MainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    MainFrame.BackgroundColor3 = Color3.fromRGB(40, 44, 52)
    MainFrame.BorderSizePixel = 0
    MainFrame.ClipsDescendants = true
    MainFrame.Parent = screenGui
    
    -- FIX: Create a UICorner object to apply rounded corners to the MainFrame
    local MainFrameCorner = Instance.new("UICorner")
    MainFrameCorner.CornerRadius = UDim.new(0, 16)
    MainFrameCorner.Parent = MainFrame
    
    -- Add a list layout to simplify arrangement
    local ListLayout = Instance.new("UIListLayout")
    ListLayout.Padding = UDim.new(0, 10)
    ListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    ListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    ListLayout.Parent = MainFrame

    -- Title Bar (LayoutOrder 1)
    local Title = Instance.new("TextLabel")
    Title.Name = "TitleBar"
    Title.Size = UDim2.new(1, 0, 0, 40)
    Title.Text = "Universal Pet Scanner & Selector"
    Title.TextColor3 = Color3.new(1, 1, 1)
    Title.TextSize = 24
    Title.Font = Enum.Font.SourceSansBold
    Title.BackgroundTransparency = 1
    Title.LayoutOrder = 1
    Title.Parent = MainFrame

    -- Close Button (Circular)
    local CloseButton = Instance.new("TextButton")
    CloseButton.Name = "CloseButton"
    CloseButton.Size = UDim2.new(0, 30, 0, 30)
    CloseButton.AnchorPoint = Vector2.new(1, 0)
    CloseButton.Position = UDim2.new(1, -10, 0, 10)
    CloseButton.Text = "✖"
    CloseButton.Font = Enum.Font.SourceSansBold
    CloseButton.TextSize = 20
    CloseButton.TextColor3 = Color3.new(1, 1, 1)
    CloseButton.BackgroundColor3 = Color3.fromRGB(255, 80, 80)
    CloseButton.AutoButtonColor = false
    CloseButton.ZIndex = 2
    CloseButton.Parent = MainFrame
    
    -- Make it circular
    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(0.5, 0) -- 50% radius makes it circular
    UICorner.Parent = CloseButton
    
    CloseButton.Activated:Connect(function()
        MainFrame.Visible = false
    end)

    -- Filter Input (LayoutOrder 2)
    local FilterBox = Instance.new("TextBox")
    FilterBox.Name = "FilterBox"
    FilterBox.Size = UDim2.new(0.9, 0, 0, 30)
    FilterBox.Text = "Filter by Species ID (e.g., 'starter_egg'). Leave empty for all."
    FilterBox.PlaceholderText = "Enter Species ID..."
    FilterBox.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
    FilterBox.TextColor3 = Color3.new(1, 1, 1)
    FilterBox.TextSize = 14
    FilterBox.Font = Enum.Font.SourceSans
    FilterBox.BackgroundColor3 = Color3.fromRGB(50, 54, 62)
    FilterBox.LayoutOrder = 2
    FilterBox.ClearTextOnFocus = true
    FilterBox.Parent = MainFrame
    
    -- NEW: Warning Label (LayoutOrder 3)
    local WarningLabel = Instance.new("TextLabel")
    WarningLabel.Name = "WarningLabel"
    WarningLabel.Size = UDim2.new(1, 0, 0, 20)
    WarningLabel.Text = "THE SCRIPT AUTO CLOSES THE XP REWARDS"
    WarningLabel.TextColor3 = Color3.fromRGB(255, 50, 50) -- Bright Red
    WarningLabel.TextSize = 14
    WarningLabel.Font = Enum.Font.SourceSansBold
    WarningLabel.BackgroundTransparency = 1
    WarningLabel.LayoutOrder = 3
    WarningLabel.Parent = MainFrame

    -- Scrolling Frame for Results (LayoutOrder 4)
    local PetListContainer = Instance.new("ScrollingFrame")
    PetListContainer.Name = "PetListContainer"
    PetListContainer.Size = UDim2.new(0.95, 0, 1, -130) -- Adjusted size to reserve more space
    PetListContainer.BackgroundTransparency = 1
    PetListContainer.CanvasSize = UDim2.new(0, 0, 0, 0) -- Will be updated dynamically
    PetListContainer.BorderSizePixel = 0
    PetListContainer.LayoutOrder = 4
    PetListContainer.Parent = MainFrame
    
    local PetListLayout = Instance.new("UIListLayout")
    PetListLayout.Padding = UDim.new(0, 5)
    PetListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Left
    PetListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    PetListLayout.Parent = PetListContainer

    -- Loading/Status Label (LayoutOrder 5)
    local StatusLabel = Instance.new("TextLabel")
    StatusLabel.Name = "StatusLabel"
    StatusLabel.Size = UDim2.new(1, 0, 0, 20)
    StatusLabel.Text = "Initializing UI..."
    StatusLabel.TextColor3 = Color3.new(0.7, 0.7, 0.7)
    StatusLabel.TextSize = 12
    StatusLabel.Font = Enum.Font.SourceSans
    StatusLabel.BackgroundTransparency = 1
    StatusLabel.LayoutOrder = 5
    StatusLabel.Parent = MainFrame

    return MainFrame, FilterBox, PetListContainer, StatusLabel, PetListLayout
end

-- ====================================================================
-- || DATA AND LOGIC FUNCTIONS ||
-- ====================================================================

-- Function to handle the selection and fire the remote server call (AddPet)
local function selectPetAndFireServer(uniqueId, cardFrame)
    local originalColor = cardFrame.BackgroundColor3
    
    -- Visual feedback for processing (yellow)
    cardFrame.BackgroundColor3 = Color3.fromRGB(255, 200, 50) 
    
    -- Structure the argument for AddPet
    local args = { 
        uniqueId
    } 
    -- Fire the remote server call with the argument unpacked
    AddPetRemote:FireServer(unpack(args))
    
    -- Simple visual feedback for user
    task.wait(0.2)
    cardFrame.BackgroundColor3 = Color3.fromRGB(80, 200, 80) -- Green flash for attempt
    task.wait(0.5)
    
    -- Revert color
    cardFrame.BackgroundColor3 = originalColor
end

-- Function to handle the removal and fire the remote server call (RemovePet)
local function removePetAndFireServer(uniqueId, cardFrame)
    local originalColor = cardFrame.BackgroundColor3
    
    -- Visual feedback for processing (orange/red)
    cardFrame.BackgroundColor3 = Color3.fromRGB(255, 100, 50) 
    
    -- Structure the argument for RemovePet
    local args = { 
        uniqueId -- Puts the selected pet's unique ID here
    } 
    -- Fire the remote server call for REMOVAL
    RemovePetRemote:FireServer(unpack(args))
    
    -- Simple visual feedback for user
    task.wait(0.2)
    cardFrame.BackgroundColor3 = Color3.fromRGB(200, 80, 80) -- Red flash for attempt
    task.wait(0.5)
    
    -- Revert color
    cardFrame.BackgroundColor3 = originalColor
end


local function createPetCard(playerName, speciesId, uniqueId, isMatch, layoutOrder)
    local Card = Instance.new("Frame")
    Card.Name = "PetCard"
    Card.Size = UDim2.new(1, 0, 0, 40)
    Card.BackgroundTransparency = 0
    Card.BackgroundColor3 = isMatch and Color3.fromRGB(60, 80, 60) or Color3.fromRGB(50, 54, 62) -- Highlight matches
    Card.LayoutOrder = layoutOrder
    Card.Parent = nil -- Parented later

    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(0, 8)
    UICorner.Parent = Card
    
    local Padding = Instance.new("UIPadding")
    Padding.PaddingLeft = UDim.new(0, 10)
    Padding.PaddingRight = UDim.new(0, 10)
    Padding.Parent = Card
    
    -- Pet Name / Species ID / UNIQUE ID Label (55% width)
    local PetNameLabel = Instance.new("TextLabel")
    PetNameLabel.Name = "PetName"
    PetNameLabel.Size = UDim2.new(0.55, 0, 1, 0) -- Adjusted width for buttons
    PetNameLabel.Position = UDim2.fromScale(0, 0)
    PetNameLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local displaySpecies = (isMatch and "⭐ " or "") .. speciesId
    PetNameLabel.Text = displaySpecies .. " | " .. uniqueId 
    PetNameLabel.TextColor3 = Color3.new(1, 1, 1)
    PetNameLabel.TextSize = 12 
    PetNameLabel.Font = Enum.Font.SourceSans
    PetNameLabel.BackgroundTransparency = 1
    PetNameLabel.Parent = Card

    -- Player Name Label (15% width)
    local PlayerNameLabel = Instance.new("TextLabel")
    PlayerNameLabel.Name = "PlayerName"
    PlayerNameLabel.Size = UDim2.new(0.15, 0, 1, 0)
    PlayerNameLabel.Position = UDim2.fromScale(0.55, 0) -- Adjusted position
    PlayerNameLabel.TextXAlignment = Enum.TextXAlignment.Center
    PlayerNameLabel.Text = playerName
    PlayerNameLabel.TextColor3 = Color3.fromRGB(150, 150, 150)
    PlayerNameLabel.TextSize = 10 
    PlayerNameLabel.Font = Enum.Font.SourceSans
    PlayerNameLabel.BackgroundTransparency = 1
    PlayerNameLabel.Parent = Card
    
    -- Select Button (12% width)
    local SelectButton = Instance.new("TextButton")
    SelectButton.Name = "SelectButton"
    SelectButton.Size = UDim2.new(0.12, 0, 0.8, 0)
    SelectButton.AnchorPoint = Vector2.new(0, 0.5)
    SelectButton.Position = UDim2.fromScale(0.71, 0.5) -- New position for 1st button
    SelectButton.Text = "SELECT"
    SelectButton.Font = Enum.Font.SourceSansBold
    SelectButton.TextSize = 10 -- Smaller font to fit
    SelectButton.TextColor3 = Color3.new(1, 1, 1)
    SelectButton.BackgroundColor3 = Color3.fromRGB(50, 120, 200) -- Blue selection color
    
    local SelectCorner = Instance.new("UICorner")
    SelectCorner.CornerRadius = UDim.new(0, 6)
    SelectCorner.Parent = SelectButton
    
    SelectButton.Activated:Connect(function()
        selectPetAndFireServer(uniqueId, Card)
    end)
    
    SelectButton.Parent = Card
    
    -- NEW: Remove Button (12% width)
    local RemoveButton = Instance.new("TextButton")
    RemoveButton.Name = "RemoveButton"
    RemoveButton.Size = UDim2.new(0.12, 0, 0.8, 0)
    RemoveButton.AnchorPoint = Vector2.new(0, 0.5)
    RemoveButton.Position = UDim2.fromScale(0.85, 0.5) -- Positioned next to Select button
    RemoveButton.Text = "REMOVE"
    RemoveButton.Font = Enum.Font.SourceSansBold
    RemoveButton.TextSize = 10 -- Smaller font to fit
    RemoveButton.TextColor3 = Color3.new(1, 1, 1)
    RemoveButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- Red remove color
    
    local RemoveCorner = Instance.new("UICorner")
    RemoveCorner.CornerRadius = UDim.new(0, 6)
    RemoveCorner.Parent = RemoveButton
    
    RemoveButton.Activated:Connect(function()
        removePetAndFireServer(uniqueId, Card)
    end)
    
    RemoveButton.Parent = Card

    return Card
end

local function updatePetList(serverData, container, listLayout, statusLabel, filterText)
    -- Clear existing results
    for _, child in ipairs(container:GetChildren()) do
        -- Only destroy frames (the pet cards)
        if child:IsA("Frame") and child.Name == "PetCard" then
            child:Destroy()
        end
    end

    local filterLower = filterText:lower()
    local totalPetsDisplayed = 0
    local layoutOrder = 0
    
    for playerName, playerData in pairs(serverData) do
        if playerData and playerData.inventory and playerData.inventory.pets then
            local playerPets = playerData.inventory.pets
            
            for uniqueId, petData in pairs(playerPets) do
                local speciesId = petData.id 
                local speciesLower = speciesId:lower()
                
                -- Check if the pet species is one of the target IDs
                local isMatch = false
                for _, targetId in ipairs(TARGET_SPECIES_IDS) do
                    if speciesId == targetId then
                        isMatch = true
                        break
                    end
                end

                -- Check against the user's filter text
                local passesFilter = true
                if filterLower ~= "" then
                    -- Filter checks if the species ID starts with the input text
                    if not speciesLower:find("^" .. filterLower) then
                        passesFilter = false
                    end
                end

                -- Only display if it passes the user's filter
                if passesFilter then
                    layoutOrder = layoutOrder + 1
                    local card = createPetCard(playerName, speciesId, uniqueId, isMatch, layoutOrder)
                    card.Parent = container
                    totalPetsDisplayed = totalPetsDisplayed + 1
                end
            end
        end
    end
    
    -- Update the Status Label and Canvas Size
    statusLabel.Text = "Displayed " .. totalPetsDisplayed .. " pets. Filter: '" .. filterText .. "'"
    
    -- FIX: Replaced the invalid 'listLayout:Wait()' call with 'task.wait()'.
    task.wait() 
    
    container.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y)
end

-- ====================================================================
-- || MAIN EXECUTION ||
-- ====================================================================

local function runScanner()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "PetSelectorScreen"
    screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

    local MainFrame, FilterBox, PetListContainer, StatusLabel, PetListLayout = createUI(screenGui)
    
    -- 1. Start the passive XP loop immediately
    startXpClaimLoop()
    
    -- 2. Start the Auto-Close loop for Level-Up UI
    startAutoCloseLoop()

    -- 3. Load Data
    StatusLabel.Text = "Fetching live server data..."
    local serverData = waitForData()
    StatusLabel.Text = "Data loaded. Initializing list..."
    
    -- 4. Initial List Population
    updatePetList(serverData, PetListContainer, PetListLayout, StatusLabel, "")

    -- 5. Filter Listener
    FilterBox.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            -- Re-run the update when the user presses Enter
            updatePetList(serverData, PetListContainer, PetListLayout, StatusLabel, FilterBox.Text)
        end
    end)
    
    FilterBox.Text = "" -- Clear default text after initial load
end

-- Run the main function
runScanner()
