# Frontier Enemy AI Behaviour

Every monster has a bunch of base command tables based on the monster ID and map ID. The game loads a bunch of base tables, then a bunch of behavior tables (area/unko/ikari) and a bunch of subtables for attack/fly/move sequences. Then Frontier does its thing and overrides each of those base tables with dozens of random tables all over the place based on events/variants/quest rank/quest ID.

## What does it look like

The command tables describe a basic ISA or ASM-like interpreted language. Each instruction starts with an opcode mapping to a specific function in the main handler; each opcode can have one or more parameters attached to it. There are logical families of opcodes (IF/ELSE, SWITCH), control flow with returns (JUMP/CALL), setters, random chance selectors, range based checks... 
To parse through a command table, given the lengths of every opcode, you simply walk through it and jump by opcode length till we find a return (opcode 0xFF) at depth = 0.

## in Code flow

The game keeps track of a program counter; every frame, the program checks where we are in the current sequence, executes the current opcode handler, and then advances to the next opcode, up to 1000 commands per frame. The main state changes happen through command 0x05 em_cmd_act_set, which dispatches a main_state and a sub_state and tells the monster to come back specifically to this point in the execution flow once it's done with the action, essentially yielding/corouting behavior.

## Example Rathian base sequence 

This is the raw data for Rathian, Jungle base sequence : 

```
0000  79 00 01 04 79 01 01 39  00 0b 00 00 05 00 06 00
0010  0b 01 80 00 03 80 01 12  05 03 06 00 80 02 04 05
0020  03 00 00 80 03 0a 05 03  0f 00 80 ff 0b 02 ff 00
0030  39 02 79 02 39 00 0b 00  00 05 00 06 00 0b 01 80
0040  00 03 80 01 10 05 00 11  00 80 02 08 05 03 06 00
0050  80 03 08 05 03 03 00 80  ff 0b 02 ff 00 39 02 79
0060  03 1b 00 01 0c 04 01 81  07 2b 00 04 01 ff 00 2b
0070  02 1b 02 0b 00 00 07 01  0b 01 07 02 0b 02 ff 00
```

 If we walk through it, accounting for opcode length and blocks we have something like that : 
 ```
79 00 01 04 79 01 01      # unique_sel ; SWITCH header
  39 00                   # eye_dmg_ck ; IF
    0b 00 00              # mode_ck ; IF
      05 00 06 00         # act_set
    0b 01                 # mode_ck ; ELSE
      80 00 03            # rnd32; RND header
        80 01 12          # rnd32; weight
          05 03 06 00     # act_set
        80 02 04          # rnd32 ; weight
          05 03 00 00     # act_set
        80 03 0a          # rnd32 ; weight
          05 03 0f 00     # act_set
      80 ff               # rnd32 ; RND end
    0b 02                 # mode_ck ; ENDIF
    ff 00                 # END
  39 02                   # eye_dmg_ck ; ENDIF
 79 02                    # unique_sel ; default
   39 00                  # eye_dmg_ck ; IF
     0b 00 00             # mode_ck ; IF
       05 00 06 00        # act_set
     0b 01                # mode_ck ; ELSE
       80 00 03           # rnd32 ; RND header
         80 01 10         # rnd32 ; weight
           05 00 11 00    # act_set
         80 02 08         # rnd32 ; weight
           05 03 06 00    # act_set
         80 03 08         # rnd32 ; weight
           05 03 03 00    # act_set
       80 ff              # rnd32 ; RND end
     0b 02                # mode_ck ; ENDIF
     ff 00                # END
   39 02                  # eye_dmg_ck ; ENDIF
79 03                     # unique_sel ; SWITCH end
1b 00 01                  # mind_ck ; IF
  0c 04 01                # flag_set
  81 07                   # call contents
  2b 00 04 01             # flag_ck ; IF
    ff 00                 # END
  2b 02                   # flag_ck ; ENDIF
1b 02                     # mind_ck ; ENDIF
0b 00 00                  # mode_ck ; IF
  07 01                   # main_jump
0b 01                     # mode_ck ; ELSE
  07 02                   # main_jump
0b 02                     # mode_ck ; ENDIF
ff 00                     # END
```

With a basic decoder we can make the flow more readable : 

```
  SWITCH unique(q=4):
      case 1:
          IF eye_damaged:
              IF mode == 0:
                  ACT 0, 6   ; yield
              ELSE:
                  RND32:
                      weight 18:
                          ACT 3, 6   ; yield
                      weight 4:
                          ACT 3, 0   ; yield
                      weight 10:
                          ACT 3, 15   ; yield
                  ENDRND
              ENDIF
              END
          ENDIF
      DEFAULT:
          IF eye_damaged:
              IF mode == 0:
                  ACT 0, 6   ; yield
              ELSE:
                  RND32:
                      weight 16:
                          ACT 0, 17   ; yield
                      weight 8:
                          ACT 3, 6   ; yield
                      weight 8:
                          ACT 3, 3   ; yield
                  ENDRND
              ENDIF
              END
          ENDIF
  ENDSWITCH
  IF mind == 1:
      flag_set 04 01
      CALL area[7]
      IF flag[4] == 1:
          END
      ENDIF
  ENDIF
  IF mode == 0:
      JMP  main[1]
  ELSE:
      JMP  main[2]
  ENDIF
  END
```

This is what it would look like with a fancier decoder 

```
fn main_0() {                  // main[0] @ 0x101452cc, 128 bytes
  switch (unique(q=4)) {
    1 => {
      if (eye_damaged) {
        if (mode == 0) {
          act(ACT, 6, 0)  // em_act06
        } else {
          random /* roll 0..31 */ {
            weight 18 => {
              act(ATTACK, 6, 0)  // em_atk06
            }
            weight 4 => {
              act(ATTACK, 0, 0)  // em_atk00
            }
            weight 10 => {
              act(ATTACK, 15, 0)  // em_atk15
            }
          }
        }
        return 0  // main (restart base[0])
      }
    }
    default => {
      if (eye_damaged) {
        if (mode == 0) {
          act(ACT, 6, 0)  // em_act06
        } else {
          random /* roll 0..31 */ {
            weight 16 => {
              act(ACT, 17, 0)  // em_act17
            }
            weight 8 => {
              act(ATTACK, 6, 0)  // em_atk06
            }
            weight 8 => {
              act(ATTACK, 3, 0)  // em_atk03
            }
          }
        }
        return 0  // main (restart base[0])
      }
    }
  }
  if (mind == 1) {
    flag_set(0x04, 0x01)
    area_7()
    if (flag[4] == 1) {
      return 0  // main (restart base[0])
    }
  }
  if (mode == 0) {
    jump main_1()
  } else {
    jump main_2()
  }
  return 0  // main (restart base[0])
}
```

## OP Codes

| op   | name                           | len     | example                       | notes                                                                                                                                                  |
| ---- | ------------------------------ | ------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0x00 | em_cmd_reset                   | 1       | `00`                          | terminator/fallback just resets stuff                                                                                                                  |
| 0x01 | em_cmd_kehai_ck                | 2       | `01 00 … 01 02`               | if/else/endif block, check if number of bit set in +2820 (detect_mask) is >0 (for player count)                                                        |
| 0x02 | em_cmd_ninshiki_ck             | 2       | `02 00 … 02 02`               | if/else/endif block, check +2687 != 0, ninshiki = "awareness", if 0 target_candidate +2612 = -1 and skip block                                         |
| 0x03 | em_cmd_area_move_ck            | 2       | `03 00 … 03 02`               | if/else/endif block, check next_area (+2046) is valid (not current stage, not 0xFFFF, check if player can reach stage) false clears a bunch of stuff   |
| 0x04 | top                            | 1       | `04`                          | used when the sequence need to go back to the top                                                                                                      |
| 0x05 | em_cmd_act_set                 | 4       | `05 03 00 00`                 | make monster do this action, return when its done, first param is main state, second param is sub state, third param is flags                          |
| 0x06 | em_cmd_target_set              | 4/5     | `06 00 00 00`                 | set target stuff target_active (+2581), target_substate (+2582), target_player (+2584) mode 0/1/2 = player targeting, mode 3 = stage targeting (short) |
| 0x07 | em_cmd_main_jmp                | 2       | `07 00`                       | basically a goto, go to where it's pointed without storing stuff for a return                                                                          |
| 0x08 | em_cmd_stand_ck                | 2       | `08 00 … 08 02`               | if/else/endif block check if (+1040) == 0                                                                                                              |
| 0x09 | em_cmd_fly_ck                  | 2       | `09 00 … 09 02`               | if/else/endif block check if (+1040) == 2                                                                                                              |
| 0x0A | em_cmd_body_status_set         | 2       | `0A 01`                       | set body_status (+2088) = param                                                                                                                        |
| 0x0B | em_cmd_mode_ck                 | 1/2/3   | `0B 00 01 … 0B 02`            | if/else/endif block, check em->mode (+2680) == param, mode is in-out of combat (0/1)                                                                   |
| 0x0C | em_cmd_flag_set                | 3       | `0C 01 01`                    | set flag, first param is which (0 = all flags 1..9), second is the value                                                                               |
| 0x0D | em_cmd_flag_clear              | 2       | `0D 01`                       | clear a flag (writes 0), param is which (0 = all)                                                                                                      |
| 0x0E | em_cmd_stage_no_ck             | 4/2/2   | `0E 00 00 05 … 0E 02`         | if/else/endif block, check stage_id (+2040) == the short area id param                                                                                 |
| 0x0F | em_cmd_route_set               | 6       | `0F 00 00 00 00 00`           | set up a movement route, look up the area table and build the route ptr, write it to +2588                                                             |
| 0x10 | em_cmd_route_ck                | 1       | `10`                          | step to the next route node, route_mode (+2592): 0 random, 1 forward, 2 wrap, >=3 hold                                                                 |
| 0x11 | em_cmd_kehai_pl_set            | 1       | `11`                          | target_player (+2584) = highest bit set in kehai_pl_mask (+2821), active=1                                                                             |
| 0x12 | em_cmd_find_ck                 | 1       | `12`                          | set target_player and target_candidate (+2612) = highest bit in perceived_mask (ninshiki) (+2687)                                                      |
| 0x13 | em_cmd_pl_target_set           | 1       | `13`                          | target_player = target_candidate (+2612) & 0xF (or -1)                                                                                                 |
| 0x14 | em_cmd_angle_ck                | 3/2/2   | `14 00 5A … 14 02`            | if/else/endif block, check the angle from em facing to the target player >= param degrees                                                              |
| 0x15 | em_cmd_stage_no_sel            | 2/4/7   | `15 00 02 … 15 03`            | switch on stage_id (+2040), cases are short stage ids                                                                                                  |
| 0x16 | em_cmd_action_set              | 2       | `16 00`                       | call into action table [n], set the return at +2608                                                                                                    |
| 0x17 | em_cmd_area_route_set          | 5       | `17 00 00 00 00`              | set up a multi-area route, no-op if area_route_counter (+2844) is running                                                                              |
| 0x18 | em_cmd_area_route_move         | 1       | `18`                          | area transition, if the counter > 1 and the next stage exists jump into the area_move table                                                            |
| 0x19 | em_cmd_area_route_ck           | 1       | `19`                          | read the area-route dest (+3208) and commit it as the target, active=3                                                                                 |
| 0x1A | em_cmd_escape_area_set         | 3       | `1A 00 05`                    | set an escape dest, active=3, area = Stage_no_at_quest of the short param                                                                              |
| 0x1B | em_cmd_mind_ck                 | 1/2/3   | `1B 00 02 … 1B 02`            | if/else/endif block, check em->mind (+2681) == param, mind = behaviour state                                                                           |
| 0x1C | em_cmd_vital_sel               | 6/3/2   | `1C 00 02 … 1C 03`            | switch on HP% (100\*vital/maxvital), cases are ascending HP%, first one <= runs                                                                        |
| 0x1D | em_cmd_mind_sel                | 6/3/2   | `1D 00 02 … 1D 03`            | switch on mind_sel_val (+2682)                                                                                                                         |
| 0x1E | em_cmd_mind_move_end           | 1       | `1E`                          | end a mind-move, clear +2681, +2682, flag_2736 and the mind state cluster                                                                              |
| 0x1F | em_cmd_sit_ck                  | 2       | `1F 00 … 1F 02`               | if/else/endif block, check posture_state (+1040) != 1                                                                                                  |
| 0x20 | em_cmd_pl_ang_sel              | 6/3/2   | `20 00 02 … 20 03`            | switch on the angle from em heading to the target player, skips if no valid player target                                                              |
| 0x21 | em_cmd_thirst_ck               | 2/2/1   | `21 00 … 21 02`               | if/else/endif block, body runs when thirst_accum (+2696) <= rate (+2708)\*0.3, over that goes to the drink branch                                      |
| 0x22 | em_cmd_near_pos_ck             | 3/2/2   | `22 00 03 … 22 02`            | if/else/endif block, check dist to horm_pos (+2852, home?) <= max(param\*100, wall_range+10)                                                           |
| 0x23 | em_cmd_emtype_sel              | 6/3/2   | `23 00 02 … 23 03`            | scan the stage enemy array, a case matches if any enemy of group <grp> is on em stage                                                                  |
| 0x24 | em_cmd_repeat_cnt_set          | 3/2/1   | `24 00 03 … 24 01 … 24 02`    | counted loop, `24 00 count` start, `24 01` --count, loops while >0, `24 02` end marker                                                                 |
| 0x25 | em_cmd_repeat_cnt_clr          | 1       | `25`                          | zero the repeat counter (+2623) to break the loop early                                                                                                |
| 0x26 | em_cmd_demo_flag_set           | 2       | `26 01`                       | set demo_flag (+2738), cutscene stuff                                                                                                                  |
| 0x27 | em_cmd_type_sel                | 6/3/2   | `27 00 02 … 27 03`            | switch on type_sel_subject (+27), case runs when subject <= thr                                                                                        |
| 0x28 | em_cmd_all_pl_same_stage_ck    | 2       | `28 00 … 28 02`               | if/else/endif block, at least one player alive and on em stage                                                                                         |
| 0x29 | em_cmd_stay_timer_ck           | 2       | `29 00 … 29 02`               | if/else/endif block, check stay_timer (+2910) <= 0 (expired)                                                                                           |
| 0x2A | em_cmd_runaway_timer_ck        | 2       | `2A 00 … 2A 02`               | if/else/endif block, check runaway_timer (+2912) <= 0 (expired)                                                                                        |
| 0x2B | em_cmd_flag_ck                 | 1/2/4   | `2B 00 01 01 … 2B 02`         | if/else/endif block, check flag[which] == val                                                                                                          |
| 0x2C | em_cmd_myemtype_sel            | 6/3/2   | `2C 00 02 … 2C 03`            | switch on em group membership, case runs if em_grp_type_ck(monster_id, grp)                                                                            |
| 0x2D | em_cmd_smell_set               | 1       | `2D`                          | smell-source target, active=7                                                                                                                          |
| 0x2E | em_cmd_search_data_set         | 2       | `2E 00`                       | pick a search data block by variant flag, store the ptr at +1968                                                                                       |
| 0x2F | em_cmd_egg_ck                  | 2       | `2F 00 … 2F 02`               | if/else/endif block, any player carrying an egg (scans the player array)                                                                               |
| 0x30 | em_cmd_egg_cancel_ck           | 1       | `30`                          | same egg scan, if anyone has an egg set egg_cancel_req (+3190) = 1                                                                                     |
| 0x31 | em_cmd_yobi_pos_set            | 1       | `31`                          | active=8, pick yobi_pos (+2864) as the move target, yobi = standby?                                                                                    |
| 0x32 | em_cmd_body_status_ck          | 3/2/2   | `32 00 01 … 32 02`            | if/else/endif block, check body_status (+2088) == param                                                                                                |
| 0x33 | em_cmd_body_status_sel         | 6/3/2   | `33 00 02 … 33 03`            | switch on body_status (+2088), exact match cases                                                                                                       |
| 0x34 | em_cmd_act_st_ck               | 4/2/2   | `34 00 03 00 … 34 02`         | if/else/endif block, check if in action main == p1 and sub == p2 right now (+21 / +20)                                                                 |
| 0x35 | em_cmd_ikari_ck                | 2       | `35 00 … 35 02`               | if/else/endif block, check ikari_state (+2726) != 0, ikari = enraged                                                                                   |
| 0x36 | em_cmd_near_pos2_ck            | 3/2/2   | `36 00 03 … 36 02`            | if/else/endif block, dist to home (+2852) <= param\*100                                                                                                |
| 0x37 | em_cmd_kunren_ck               | 2       | `37 00 … 37 02`               | if/else/endif block, kunren = training, check the training quest flag (quest_struct[11] & 0x200000)                                                    |
| 0x38 | em_cmd_water_ck                | 2       | `38 00 … 38 02`               | if/else/endif block, does this area have water (GetWaterData)                                                                                          |
| 0x39 | em_cmd_eye_dmg_ck              | 2       | `39 00 … 39 02`               | if/else/endif block, eyes damaged (+2914 != 0), flash bomb behaviour gate                                                                              |
| 0x3A | em_cmd_sensor_ck               | 2       | `3A 00 … 3A 02`               | if/else/endif block, check flag_3186 (+3186) != 0                                                                                                      |
| 0x3B | em_cmd_boss_work_ck            | 2       | `3B 00 … 3B 02`               | if/else/endif block, check boss_em_ptr (+3172) != 0, drome boss behaviour?                                                                             |
| 0x3C | em_cmd_boss_atk_ck             | 2       | `3C 00 … 3C 02`               | if/else/endif block, boss present and attacking (boss->mode == 1) and on em stage                                                                      |
| 0x3D | em_cmd_before_stage_ck         | 4/2/2   | `3D 00 00 05 … 3D 02`         | if/else/endif block, before_stage (+2846) == the short param                                                                                           |
| 0x3E | em_cmd_before_stage_sel        | 7/4/2   | `3E 00 02 … 3E 03`            | switch on before_stage (+2846), cases are short stage ids                                                                                              |
| 0x3F | em_cmd_ground_area_move        | 4       | `3F 00 00 05`                 | pick the nearest reachable node toward a dest area and setup the move                                                                                  |
| 0x40 | em_cmd_em_mode_change          | 2       | `40 01`                       | set mode via Em_Mode_Chg(em, v != 0), 0/1                                                                                                              |
| 0x41 | em_cmd_em_hp_vital_add         | 2       | `41 00`                       | HP regen for the drome leaders only, Velocidrome heals 15%, Gendrome/Iodrome 10%                                                                       |
| 0x42 | em_cmd_horm_pos_ang_ck         | 3/2/2   | `42 00 5A … 42 02`            | if/else/endif block, check if facing more than param degrees away from home (horm_pos +2852)                                                           |
| 0x44 | em_cmd_boss_same_stage_ck      | 2       | `44 00 … 44 02`               | if/else/endif block, boss exists and is on em stage                                                                                                    |
| 0x45 | em_cmd_pl_fishing_ck           | 2       | `45 00 … 45 02`               | if/else/endif block, is a player fishing em with frog bait (the Plesioth thing)                                                                        |
| 0x46 | em_cmd_target_pl_act_ck        | 4/2/1   | `46 00 03 00 … 46 02`         | if/else/endif block, is the target player doing action main/sub right now                                                                              |
| 0x47 | em_cmd_fish_ok_ck              | 2       | `47 00 … 47 02`               | if/else/endif block, fish_ok_latch (+2729) != 0, clears it on true (one-shot)                                                                          |
| 0x48 | em_cmd_timer_set               | 2       | `48 1E`                       | set cmd_timer (+3228) = param                                                                                                                          |
| 0x49 | em_cmd_pl_land_target          | 2       | `49 00`                       | target a player's landing spot, set target_mode (+3231)/active = 11                                                                                    |
| 0x4A | em_cmd_pl_look_ck              | 2       | `4A 00 … 4A 02`               | if/else/endif block, is the target player looking?? (bit in pl_look_mask +2684)                                                                        |
| 0x4B | em_cmd_kehai_clear             | 1       | `4B`                          | zero the current target's kehai/presence value (per-player float array +2788)                                                                          |
| 0x4C | em_cmd_hate_clear              | 1       | `4C`                          | zero the current target's hate (hate_table +2824)                                                                                                      |
| 0x4D | em_cmd_horm_pos_set            | 1       | `4D`                          | store my current pos into horm_pos (+2852)                                                                                                             |
| 0x4E | em_cmd_thirst_add              | 2       | `4E 00`                       | add to thirst (+2696 += rate/2)                                                                                                                        |
| 0x4F | em_cmd_hungry_add              | 2       | `4F 00`                       | add to hunger (+2700 += rate/2)                                                                                                                        |
| 0x50 | em_cmd_suimin_add              | 2       | `50 00`                       | add to sleep (suimin = sleep, +2704 += rate/2)                                                                                                         |
| 0x51 | em_cmd_swim_ck                 | 2       | `51 00 … 51 02`               | if/else/endif block, check posture_state (+1040) == 4                                                                                                  |
| 0x52 | em_cmd_all_pl_target_sel       | 1       | `52`                          | pick target_candidate among all perceived players, hate tiers 50k/30k/any                                                                              |
| 0x53 | em_cmd_samestage_pl_target_sel | 1       | `53`                          | pick target_candidate among same-stage perceived players, hate tiers 50k/30k/any, specific stuff for Orugaron                                          |
| 0x54 | em_cmd_target_pl_samestage_ck  | 2       | `54 00 … 54 02`               | if/else/endif block, target exists and same-stage and not flagged                                                                                      |
| 0x55 | em_cmd_target_pl_hate_high_ck  | 2       | `55 00 … 55 02`               | if/else/endif block, target player hate >= 30000                                                                                                       |
| 0x56 | em_cmd_quest_no_ck             | 4/2/2   | `56 00 00 05 … 56 02`         | if/else/endif block, current quest id == the short param                                                                                               |
| 0x57 | em_cmd_move_tbl_no_sel         | 3/4/2   | `57 00 02 … 57 03`            | switch on move_tbl_no (+3185), cases are short thresholds, first >= subject wins                                                                       |
| 0x58 | em_cmd_boss_pl_target_set      | 1       | `58`                          | copy the boss's target player                                                                                                                          |
| 0x59 | em_cmd_tenjo_ck                | 2       | `59 00 … 59 02`               | if/else/endif block, tenjo_state (+3200) == 0, tenjo = ceiling                                                                                         |
| 0x5A | em_cmd_target_land_no_ck       | 3/2/1   | `5A 00 01 … 5A 02`            | if/else/endif block, is the target on land/floor == param                                                                                              |
| 0x5B | em_cmd_ninshiki_timer_sub      | 2       | `5B 00`                       | reset recognition timers (+2688) for players not perceived                                                                                             |
| 0x5C | em_cmd_tenjostage_ck           | 2       | `5C 00 … 5C 02`               | if/else/endif block, on a ceiling stage and the ceiling raycast hits                                                                                   |
| 0x5D | em_cmd_smell_set_ck            | 2       | `5D 00 … 5D 02`               | if/else/endif block, commit the home/smell pos, body runs if it's a non-zero point                                                                     |
| 0x5E | em_cmd_my_floor_ck             | 3/2/2   | `5E 00 01 … 5E 02`            | if/else/endif block, check floor/height layer == param (computed from position)                                                                        |
| 0x5F | em_cmd_st25_pl_target_sel      | 1       | `5F`                          | stage-25 (Fatalis/Schrade) target picker, filter by floor then hate tier                                                                               |
| 0x60 | em_cmd_st25_gate_ck            | 2       | `60 00 … 60 02`               | if/else/endif block, Fatalis st25 gate global (game_manager[9466]) != 0                                                                                |
| 0x61 | em_cmd_runaway_timer_set       | 1       | `61`                          | set runaway_timer (+2912) from the per monster reload table                                                                                            |
| 0x62 | em_cmd_dansa_sel               | 2       | `62 00 … 62 ff`               | selector on elevation (dansa = step/height diff), picks a case by value bands, Fatalis Schrade                                                         |
| 0x63 | em_cmd_type_and_ck             | 3/2/2   | `63 00 04 … 63 02`            | if/else/endif block, check (type_sel_subject +27 & mask) != 0                                                                                          |
| 0x64 | em_cmd_target_pl_hate_ck       | 3/2/1   | `64 00 00 … 64 02`            | if/else/endif block, target hate >= global threshold table[param]                                                                                      |
| 0x65 | em_cmd_all_pl_hate_clear       | 1       | `65`                          | zero the 4 hate entries (+2824)                                                                                                                        |
| 0x66 | em_cmd_pl_ride_ck              | 2       | `66 00 … 66 02`               | if/else/endif block, check if nobody climbed on, Lao-Shan/Shen/Yama/Raviente                                                                           |
| 0x67 | em_cmd_em_master_ck            | 2       | `67 00 … 67 02`               | if/else/endif block, am I a remote/non-master copy (Online stuff maybe?) (act_suppress +2739 != 0)                                                     |
| 0x68 | em_cmd_em_cmd_reset            | 1       | `68`                          | re-init flags but keep running the table                                                                                                               |
| 0x69 | em_cmd_horm_pos_ang_ck2        | 3/2/1   | `69 00 5A … 69 02`            | if/else/endif block, facing away from home by >= param degrees, or param >= 90                                                                         |
| 0x70 | em_cmd_myemtype_sel_0          | 6/3/2   | `70 00 02 … 70 03`            | switch on monster_id (+3)                                                                                                                              |
| 0x71 | em_cmd_nando_ck                | 3/2/1   | `71 00 02 … 71 02`            | if/else/endif block, quest difficulty (+3212) >= param                                                                                                 |
| 0x72 | em_cmd_arena_ck                | 2       | `72 00 … 72 02`               | if/else/endif block, check if arena quest (quest id 25001, online, flag)                                                                               |
| 0x73 | em_call_act_sel                | 6/3/2   | `73 00 02 … 73 03`            | switch on call_act_req (+2920) & 0x7F, the case consumes the request on a hit                                                                          |
| 0x74 | em_call_act_ck                 | 3/2/2   | `74 00 01 … 74 02`            | if/else/endif block on call_act_req, not arena skips, arena checks > thr                                                                               |
| 0x75 | em_cmd_type2_sel               | 6/3/2   | `75 00 02 … 75 03`            | switch on em_subtype (+2930)                                                                                                                           |
| 0x76 | em_cmd_season_sel              | 6/3/2   | `76 00 02 … 76 03`            | switch on the abs season number                                                                                                                        |
| 0x77 | em_cmd_day_ck                  | 2       | `77 00 … 77 02`               | if/else/endif block, the day/night global (quest_get_day_night != 0)                                                                                   |
| 0x78 | em_cmd_haba_angle_ck           | 4/2/2   | `78 00 10 40 … 78 02`         | if/else/endif block, target angle within [lo<<8, hi<<8]                                                                                                |
| 0x79 | em_cmd_unique_sel              | 2/3/7   | `79 00 02 … 79 03`            | switch on a monster-specific unique-program callback                                                                                                   |
| 0x7A | em_cmd_map_sel                 | 6/3/2   | `7A 00 02 … 7A 03`            | switch on the current map (location id, night to day)                                                                                                  |
| 0x7B | em_act_rnd_update              | 1       | `7B`                          | bump AI rng (rng_value +1096)                                                                                                                          |
| 0x7C | em_cmd_tougi_ck                | 2       | `7C 00 … 7C 02`               | if/else/endif block, current map is not an arena/tougi map                                                                                             |
| 0x7D | em_cmd_var_sel                 | 6/3/2   | `7D 00 02 … 7D 03`            | switch on monster variant, case runs if em_variation_check(monster_id, v)                                                                              |
| 0x7E | em_cmd_all_target_set          | 1       | `7E`                          | roll a probability between targeting another enemy (active=13) or a player                                                                             |
| 0x7F | em_cmd_tower_chase_ck          | 2       | `7F 00 … 7F 02`               | if/else/endif block, Tower arena chase check                                                                                                           |
| 0x80 | em_cmd_rnd32                   | 1/2/3/6 | `80 00 03  80 01 10 …  80 ff` | weighted random, roll = rng_value & 0x1F (0..31), header / weighted cases / end                                                                        |
| 0x81 | em_cmd_contents                | 2       | `81 00`                       | call into area table [n]                                                                                                                               |
| 0x82 | em_cmd_sub_contents            | 3       | `82 00 00`                    | call into the subtable, 2D index field[15+arg0][arg1]                                                                                                  |
| 0x83 | em_cmd_range_ck                | 3/2/1   | `83 00 02  83 01 …  83 ff`    | switch on distance-to-target bands, picks the band by dist vs the monster thresholds table (+2560)                                                     |
| 0x84 | em_act_rnd_update              | 1       | `84`                          | bump AI rng (rng_value +1096)                                                                                                                          |
| 0x85 | em_cmd_tower_chase_pos_set     | 1       | `85`                          | set the tower chase flag (+1042) and the next chase home pos                                                                                           |
| 0x86 | em_cmd_tower_chase_flg_off     | 1       | `86`                          | clear the tower chase flag (+1042)                                                                                                                     |
| 0x90 | em_cmd_position_set            | 2       | `90 00`                       | copy a position (monster table) into pos (+172) and +1784                                                                                              |
| 0x91 | em_cmd_vec_set                 | 2       | `91 00`                       | face the em toward a position                                                                                                                          |
| 0x92 | em_cmd_demo_start              | 1       | `92`                          | stub                                                                                                                                                   |
| 0x93 | em_cmd_wait_set                | 1       | `93`                          | stub                                                                                                                                                   |
| 0x94 | em_cmd_em_atk_bit              | 3/3/2   | `94 00 02  94 01 01 …  94 03` | switch on the global g_em_atk_select                                                                                                                   |
| 0x99 | em_cmd_vec_set2                | 2       | `99 00`                       | set heading, facing_angle (+164) = param << 8                                                                                                          |
| 0x9A | em_cmd_hungry_ck               | 3/2/1   | `9A 00 32 … 9A 02`            | if/else/endif block, body runs when hunger_accum (+2700) < param%                                                                                      |
| 0x9B | em_cmd_tgt_jump_pl_ck          | 2       | `9B 00 … 9B 02`               | if/else/endif block, is the target player in air                                                                                                       |
| 0x9C | em_cmd_tgt_jimen_height_ck     | 3/2/2   | `9C 00 01 … 9C 02`            | if/else/endif block, target on the ground near home height                                                                                             |
| 0xFF | em_cmd_end_command             | 2       | `FF 00`                       | RET / end of block (`FF 00` END/restart, `FF 01/02/03` RET to caller)                                                                                  |

## Methodology

Reversed mainly through static analysis on the PC dll of Monster Hunter Frontier Z, debug symbols validated through the WiiU dissassembly of the game.
