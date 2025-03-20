# TSMC
import networkx as nx
import matplotlib.pyplot as plt
import pandas as pd
import random

# 讀取必要的資料
wips_df = pd.read_csv('/content/TSMC/WIP.csv')  # WIP 資料，包含 WIP_ID, Remaining Q-Time, FROM, TO
df_xfer_time = pd.read_csv('/content/TSMC/XFER_TIME.csv')  # XFER_TIME 資料，包含 FROM, TO, XFER_TIME
df_xfer_time.drop(columns=['Unnamed: 3'], inplace=True)
df_cart = pd.read_csv('/content/TSMC/CART.csv')  # CART 資料，包含 CART_ID, INIT_LOC

# 讀取排程結果
schedule_df = pd.read_csv('/content/TSMC/輸入輸出/schedule_output1.csv')

# Create a directed graph
G = nx.DiGraph()

# Add edges (僅用於檢查路徑和獲取 XFER_TIME，邊不會顯示)
for _, row in df_xfer_time.iterrows():
    if row["FROM"] != row["TO"]:  # Ignore self-loops
        G.add_edge(row["FROM"], row["TO"], weight=row["XFER_TIME"])

# Draw the graph
plt.figure(figsize=(12, 10))
pos = nx.spring_layout(G, seed=42)  # Layout for visualization

# 手動調整 LOC16 和 LOC39 的位置，避免重疊
pos["LOC16"] = (pos["LOC16"][0] + 0.1, pos["LOC16"][1] + 0.1)  # LOC16 右上偏移
pos["LOC39"] = (pos["LOC39"][0] - 0.1, pos["LOC39"][1] - 0.1)  # LOC39 左下偏移

# 僅繪製節點（LOC），不繪製邊
nx.draw_networkx_nodes(G, pos, node_size=2000, node_color="lightblue")
nx.draw_networkx_labels(G, pos, font_size=10)

# 為每台推車生成隨機顏色，並映射到 WIP
cart_colors = {cart_id: "#{:06x}".format(random.randint(0, 0xFFFFFF)) for cart_id in schedule_df['CART_ID'].unique()}
wip_colors = {}  # WIP 和對應推車的顏色映射
wip_positions = {}  # WIP 的顯示位置

# 根據排程結果，確定 WIP 的新位置（第一個 DELIVERY 的 TO）與顏色
for cart_id in schedule_df['CART_ID'].unique():
    cart_schedule = schedule_df[schedule_df['CART_ID'] == cart_id].sort_values('ORDER')
    delivery_rows = cart_schedule[cart_schedule['ACTION'] == 'DELIVERY']
    pickup_rows = cart_schedule[cart_schedule['ACTION'] == 'PICKUP']

    if len(delivery_rows) >= 1 and len(pickup_rows) >= 2:  # 確保有第一個 DELIVERY 和第二個 PICKUP
        first_delivery_row = delivery_rows.iloc[0]  # 第一個 DELIVERY (ORDER=2)
        second_pickup_row = pickup_rows.iloc[1]    # 第二個 PICKUP (ORDER=3)

        wip_id_first = first_delivery_row['WIP_ID']
        wip_id_second = second_pickup_row['WIP_ID']

        # 第一個 WIP 移動到第一個 DELIVERY 的 TO 位置
        new_loc = wips_df[wips_df['WIP_ID'] == wip_id_first]['TO'].iloc[0]
        wip_positions[wip_id_first] = new_loc
        wip_colors[wip_id_first] = cart_colors[cart_id]

        # 第二個 WIP 保持其初始位置（FROM）
        wip_positions[wip_id_second] = wips_df[wips_df['WIP_ID'] == wip_id_second]['FROM'].iloc[0]
        wip_colors[wip_id_second] = cart_colors[cart_id]

# 標示所有 WIP 的位置，考慮移動後的位置
for wip_id in wips_df['WIP_ID']:
    if wip_id not in wip_positions:  # 未移動的 WIP 保持初始位置
        wip_positions[wip_id] = wips_df[wips_df['WIP_ID'] == wip_id]['FROM'].iloc[0]
        wip_colors[wip_id] = 'darkred'  # 未分配推車的 WIP 預設為暗紅色

for wip_id, loc in wip_positions.items():
    color = wip_colors[wip_id]
    plt.scatter([pos[loc][0]], [pos[loc][1]], color='red', s=100, label='WIP Start' if wip_id == list(wip_positions.keys())[0] else "")

    # 根據 LOC 位置調整標籤偏移量
    offset_x, offset_y = 0.02, 0.02  # 預設偏移量
    if loc == "LOC16":
        offset_x, offset_y = 0.03, 0.03  # LOC16 標籤右上偏移
    elif loc == "LOC39":
        offset_x, offset_y = -0.03, -0.03  # LOC39 標籤左下偏移

    # 若有多個 WIP 在同一位置，稍微偏移標籤
    same_loc_wips = [wid for wid, l in wip_positions.items() if l == loc]
    if len(same_loc_wips) > 1 and wip_id in same_loc_wips:
        index = same_loc_wips.index(wip_id)
        offset_y += 0.03 * index  # 垂直偏移，避免重疊

    plt.text(pos[loc][0] + offset_x, pos[loc][1] + offset_y, wip_id, color=color, fontsize=8, fontweight='bold')

# 繪製每台推車的 ORDER=2 移動路線（從第一個 DELIVERY 到第二個 PICKUP），並標示花費時間與車輛編號
for cart_id in schedule_df['CART_ID'].unique():
    cart_schedule = schedule_df[schedule_df['CART_ID'] == cart_id].sort_values('ORDER')

    # 提取 ORDER=2 的移動路徑（從第一個 DELIVERY 到第二個 PICKUP）
    path = []
    delivery_rows = cart_schedule[cart_schedule['ACTION'] == 'DELIVERY']
    pickup_rows = cart_schedule[cart_schedule['ACTION'] == 'PICKUP']

    if len(delivery_rows) >= 1 and len(pickup_rows) >= 2:  # 確保有第一個 DELIVERY 和第二個 PICKUP
        first_delivery_row = delivery_rows.iloc[0]  # 第一個 DELIVERY (ORDER=2)
        second_pickup_row = pickup_rows.iloc[1]    # 第二個 PICKUP (ORDER=3)

        wip_id_first = first_delivery_row['WIP_ID']
        wip_id_second = second_pickup_row['WIP_ID']

        start_loc = wips_df[wips_df['WIP_ID'] == wip_id_first]['TO'].iloc[0]  # 第一個 DELIVERY 的 TO
        end_loc = wips_df[wips_df['WIP_ID'] == wip_id_second]['FROM'].iloc[0]  # 第二個 PICKUP 的 FROM

        path.append(start_loc)
        path.append(end_loc)

        # 繪製 ORDER=2 的移動路徑，並標示花費時間與車輛編號
        if start_loc in pos and end_loc in pos:
            # 獲取花費時間（XFER_TIME）
            xfer_time = G[start_loc][end_loc]['weight'] if G.has_edge(start_loc, end_loc) else "N/A"

            # 繪製路徑（顏色與 WIP 標籤一致）
            if G.has_edge(start_loc, end_loc):
                nx.draw_networkx_edges(G, pos, edgelist=[(start_loc, end_loc)],
                                       edge_color=cart_colors[cart_id], width=2, arrows=True,
                                       label=f'Cart {cart_id}')
            else:
                # 若邊不存在，繪製虛線表示路徑
                plt.plot([pos[start_loc][0], pos[end_loc][0]], [pos[start_loc][1], pos[end_loc][1]],
                         color=cart_colors[cart_id], linestyle='--', linewidth=2,
                         label=f'Cart {cart_id} (Indirect)')

            # 在路徑中間標示花費時間與車輛編號（黑色）
            if xfer_time != "N/A":
                mid_x = (pos[start_loc][0] + pos[end_loc][0]) / 2
                mid_y = (pos[start_loc][1] + pos[end_loc][1]) / 2
                label_text = f'{xfer_time} ({cart_id})'  # 顯示 XFER_TIME 和 CART_ID
                plt.text(mid_x, mid_y, label_text, color='black', fontsize=8, fontweight='bold',
                         bbox=dict(facecolor='white', edgecolor='none', alpha=0.7))

plt.title("LOC Network with Moved WIP Positions and Second Cart Moves (Time and Cart Labeled)")
plt.legend()
plt.show()
